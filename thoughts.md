# Cluster Management

## Problem Statement

Distributed computing environments face two significant problems

* Computing resources need to be assigned to computing tasks in a coordinated way (ie, if I have two computers and two tasks, I don't want both of the computers trying to do the same task)
* Process failure needs to be detected and responded to (ie if I run out of disk space, another process should take over my task)

An ideal solution would satisfy a couple objectives

1. 'Failure' would be defined in-process (as opposed to simply asking if an instance is running)
2. Task assignment would be able to tolerate instances entering & leaving the cluster
3. There would be no single point of failure
4. It would be language-agnostic (could be used with Ruby, Java, Node, etc)
5. Task assignment could (optionally) be static unless the cluster topology changed

We've tried a couple approaches to solving this:

* Tell each computer what it's supposed to do when we provision it, and alert a human when something fails (OIB scheduler box).  This fails #'s 2 & 3 (at least for singular resources like the scheduler)
* Each instance claims a task for a duration of time and coordinate through a central data store.  Cycle assignment regularly so failures can be healed (AMP instances).  This fails #5
* Use a process management framework to do it for us (Storm topology).  This depends on the framework, but I don't know of any frameworks that satisfy all objectives.


## Proposed Solution

Consul provides a mechanism for service discovery and consistent, distributed KV store.  It provides an HTTP api which accepts blocking requests which can be used to generate event handlers when the KV store (or the cluster topology) changes.  I propose we build a simple daemon which handles cluster management on top of these capabilities.

The daemon would be configured to respond to various events:

```ruby
Manager.new do |m|
  # Execute this when the daemon first starts up
  m.bootstrap do
    # Fetch the current value and ModifyIndex of the key 'tasks'
    tasks, idx = m.kv.get("tasks")
    
    unless tasks
      # If the expected tasks haven't been defined, create them with empty values
      new_tasks = {
        "Consume Kafka Partition 1": nil,
        "Consume Kafka Partition 2": nil,
        # ...
      }
      
      # Write these to the store (assuming they still don't exist)
      m.kv.check_and_set(
        key: "tasks", 
        check: idx,
        value: new_tasks
      )
    end
  end
  
  m.bootstrap do
    tasks, idx = m.kv.get("tasks")
    
    # Try to acquire a task
    if target_task = tasks.select { |k, v| v.nil? }.shuffle.first
      m.kv.check_and_set(
        key: "tasks',
        check: idx,
        value: tasks.merge(target_task: m.node),
        on_success: ->() { `/start_my_job.sh` },  # If we acquired this key, start doing stuff
        on_failure: ->() { retry },               # Otherwise, try again
      )
    end
  end
  
  # Monitor this HTTP endpoint for changes and execute the block with the old & new responses on change
  m.on_change("/v1/health/service/kafka_consumers") do |old, new|
    old_failing = old.select { |h| h["Status"] == failing }.map { |h| h["Node"] }
    new_failing = new.select { |h| h["Status"] == failing }.map { |h| h["Node"] }
    
    # Terminate instances with failing health checks
    (new_failing - old_failing).each do |newly_failed_node|
      terminate_instance(newly_failed_node)
    end
  end
  
  m.on_change("/v1/catalog/nodes") do |old, new|
    removed_nodes = (old - new).map { |h| h["Node"] }
    
    tasks, idx = m.kv.get("tasks")
    new_tasks = tasks.hash_map { |k, v| removed_nodes.include?(v) ? nil : v }
    
    m.kv.check_and_set(
      key: "tasks",
      check: idx,
      value: new_tasks,
      on_failure: ->() { retry }  # If another process modified the key, start over
    end
  end
  
  m.teardown do
    # Gracefully remove this instance from the task assignment hash
    tasks, idx = m.kv.get("tasks")
    new_tasks = tasks.hash_map { |k, v| v == m.node ? nil : v }
    
    m.kv.check_and_set(
      key: "tasks",
      check: idx,
      value: new_tasks,
      on_failure: ->() { retry }
    end
  end
end

# Run the daemon!
Manager.run
```

Obviously these specific tasks need some work, but you get the idea of the daemon API.