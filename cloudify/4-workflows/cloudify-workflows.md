# Cloudify Workflows

## WHAT ARE WORKFLOWS?

Put simply, Cloudify workflows are programs that determine what orchestration tasks will be executed, and when. These tasks may be implemented by plugins, or by arbitrary code included in the blueprint. Workflows are written in Python, using the Cloudify framework.

Every deployment has its own set of workflows, declared in the parent blueprint. They are always executed in the context of that deployment; that is to say, when executed, they only apply to the deployment that invoked them.

## CONTROLLING WORKFLOWS

Controlling workflows, such as executing or cancelling them, is done via REST calls to the management server. Here, examples will be shown using Cloudify CLI commands which in turn call the above REST API calls.

## STARTING AN EXECUTION

This is how you'd execute a hypothetical my_workflow on the my_deployment deployment, from a properly initialized CLI:

```yaml
cfy executions start -d my_deployment -w my_workflow [-p parameters.yaml]
```

As you can infer from the above, it is also possible to feed a list of parameters into a workflow execution. These are declared in the blueprint, and they can be either mandatory or optional, with a pre-defined default. If you need to find out about a workflow's parameters, including which are optional and which are mandatory, you can issue the following command:

```yaml
cfy workflows get -d my_deployment -w my_workflow
```

## EXECUTION STATUS

When a workflow is executed, a unique execution object is created and stored, containing information about the execution run. This object stores, among other things, the status of the execution, such as "started", "cancelled", or "terminated" (the latter of which indicates success), and the parameters it was called with.

You can consult the different workflow executions of a deployment, and their respective statuses, by issuing the following command:

```yaml
cfy executions list -d my_deployment
```

To retrieve the parameters an execution was started with, note the identifier of the execution in the list above and substitute it for [EXECUTION_ID] in the following command:
```yaml
cfy executions get -e [EXECUTION_ID]
```

## BUILT-IN WORKFLOWS

Cloudify comes with a number of built-in workflows. Currently, these are install, uninstall, heal, and scale, as well as a generic workflow for executing operations called execute_operation.

Built-in workflows are declared and mapped in the blueprint in types.yaml, which is usually imported either directly or indirectly via other imports.

## BUILT-IN WORKFLOWS: INSTALL

The job of the install built-in workflow is to install an application. The application's blueprint is used by this workflow to determine where, and in what order, nodes are to be created, as well as how to implement the relationships to each other.

You will see that this workflow, as well as uninstall, is used by both the heal, and scale workflows.

## BUILT-IN WORKFLOWS: UNINSTALL

This workflow does the exact opposite of the previous one: it uninstalls an application. It does so by referring to the blueprint to identify node relationships to unlink, then the nodes to remove.

## BUILT-IN WORKFLOWS: HEAL

For a subset of a blueprint's topology, the heal workflow calls the uninstall workflow, immediately followed by install. It expects a node instance ID as a parameter.

For example, if you give it a compute node ID, it will find the VM, recreate it from scratch including any services it contains, and re-establish all applicable node relationships.

## BUILT-IN WORKFLOWS: SCALE

Like the heal workflow, the scale workflow applies only to a subset of a blueprint's topology. You can use it to increase or decrease the number of nodes given by the input node_id. You select whether to create more nodes or delete existing ones by passing in a respectively positive or negative delta paremeter.

In practice, you'll call the scale workflow either when your application needs more horsepower to service more users, or when you need to conserve resources.

## BUILT-IN WORKFLOWS: EXECUTE_OPERATION

This is a generic workflow, used when one needs to execute arbitrary operations on nodes. The operation itself may be given as the path to an executable script, or a Python module implemented by a Cloudify plugin.

When invoked, the execute_operation workflow will calculate all nodes on which to execute the custom operation, then execute it either sequentially or in parallel.