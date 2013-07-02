# Vector

Vector is a tool that augments your auto-scaling groups. Currently the
only additional feature it provides is Predictive Scaling based on
historical data.

## Predictive scaling

Auto Scaling groups do a good job of responding to current
load conditions, but if you have a predictable load pattern,
it can be nice to scale up your servers a little bit *early*.
Some reasons you might want to do that are:

 * If it takes several minutes for an instance to fully boot
   and ready itself for requests.
 * If you have very serious (but predictable) spikes,
   it's nice to have the capacity in place before the spike
   starts.
 * To give yourself a buffer of time if AWS APIs start
   throwing errors. If scaling up is going to fail, you'd
   rather it start failing with a little bit of time before
   you actually need the capacity so you can begin evasive maneuvers.

Vector examines your existing CloudWatch alarms tied to your Auto
Scaling groups, and checks those metrics in the past. It does this by
comparing the "Load" of the metric in the past versus the present. The
"Load" of a metric is defined as the metric_value *
number_of_current_instances.

Vector will first check the metric load right now and the metric load in
the past, based on the lookback window. If those are within a threshold,
we assume that the future will mirror the past and continue.

Vector then retrieves the load of the metric from the past plus the
lookahead window. That load is divided by the current number of
instances to get the "predicted load". That value is compared against
the alarm, and if it would send the alarm into ALARM state, it triggers
the scaleup policy.

This method will only work if multiplying your metric by # of instances
to come up with a representation of load across the group makes sense.
For things like CPUUtilization this is true. Other metric types should
be thought through.

## Installation

```bash
$ gem install vector
```

## Requirements

 * Auto Scaling groups must have the GroupInServiceInstances metric
   enabled.
 * Auto Scaling groups must have at least one scaling policy with a
   positive adjustment, and that policy must have at least one
   CloudWatch alarm with a CPUUtilization metric.

## Usage

Typically vector will be invoked via cron periodically (every 10 minutes
is a good choice.)

First create a configuration file:

```yaml
# Predictive scaling looks back in time, and in order to handle DST
# correctly, it needs to know what timezone you want time arithmetic to
# happen in.
timezone: America/Denver

groups:
  - my-asg-group1
  - my-asg-group2

predictive-scaling:
  # A list of windows of time to look back. CloudWatch only
  # stores data for 2 weeks, so you can't go back any further than that.
  # (required)
  lookback-windows:
    - 1w
    - 2w

  # How far ahead of time to attempt to scale up.
  # (required)
  lookahead-window: 1h

  # A value from 0 - 1.0 that specifies how closely current load
  # must match the load from $lookback_window ago in order for
  # predictive scaling to take place. This lets you only do predictive
  # scaling only if current load is similar to past load.
  # 
  # If this value is not specified, predictive scaling will take
  # place no matter what current load is.
  # (optional)
  valid-threshold: 0.8

  # The period to use when doing the threshold check from above.
  valid-period: 10m

flexible-down-scaling:
  # TODO: document cooldown periods
  up-to-down-cooldown: 45m
  down-to-down-cooldown: 10m
```

And run Vector:

```bash
$ vector -c config.yml
```

When Vector decides that a predictive scaleup should happen, it will
trigger the policy attached to the alarm that was being evaluated.

When Vector sees that a scaledown alarm is ACTIVE and the proper
cooldown period has passed, it will trigger the associated policy to
scaledown.

# Questions

### Why not just predictively scale based on the past DesiredInstances?

If we don't look at the actual utilization and just look
at how many instances we were running in the past, we will end up
scaling earlier and earlier, and will never re-adjust and not scale up
if load patterns change, and we don't need so much capacity.