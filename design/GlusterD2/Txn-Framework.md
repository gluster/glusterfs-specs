# Transaction framework

The transaction framework will ensure that a given set of actions will be performed in order, one after another, on the required peers of the cluster.

The new transaction framework will be built around the central store.
A GD2 transaction will have the following characteristics,

1. actions will only be performed where required, instead of being done across the whole cluster
2. final committing into the store will only be done by the node where the transaction was initiated, instead of being done on all nodes.


## Transaction

A transaction is basically a collection of steps or actions to be performed in order.
A transaction object provides the framework with the following,

1. a list of nodes that will be a part of the transaction
2. a set of transaction [steps](#transaction-step)

Given this information, the GD2 transaction framework will,

- verify if all the listed nodes are online
- run each step on all of the nodes, before proceeding to the next step
- if a step fails, undo the changes done by the step and all previous steps.

The base transaction is basically free-form, allow users to create any order of steps. This keeps it flexible and extensible to create complex transactions.

### Transaction step
A step is an action to be performed, most likely a function that needs to be run.
A step object provides the following information,

1. The function to be run
2. The list of nodes the step should be run on.
3. An undo function that reverts any changes done by the step.

Each step can have its own list of nodes, so that steps can be targeted to specific nodes and provide more flexibility. The list of nodes can also be specified as ALL or LEADER (Leader indicates the originator node of the transaction) to target all nodes in the transaction or just the leader.

## Simple transaction template

Any user is free to create free-form transactions by providing their own set of steps.
To make it easier to quickly create simple transactions, a simple transaction template will be provided.

The template will be based on the existing four step transaction algorithm being used.
The template requires users to provide the following information

1. a list of nodes to run the transaction on
2. an object within the GD2 store to obtain a lock on
3. a staging function, which checks is the node can perform the transaction
4. a perform function, which performs the transaction action on the node
5. a rollback function, to undo changes done by the commit function
6. a store function to save results if required in to the store

When following the common template, a transaction will occur as follows.

1. the transaction framework verifies all listed nodes are online
2. the initiator node obtains a lock on the specified object
3. the staging function is run on the listed nodes to verify if the transaction can happen
4. the perform function is run on the listed nodes to actually perform the operation
5. the store function is run on the initiator to store results
6. the initiator node unlocks the locked object

If any of the above fail (except unlock) fail, the transaction is aborted.
- If 1 or 2 fail, the transaction is aborted immediately.
- If 3 fails, unlock is done and transaction is aborted.
- If 4 or 5 fail, the rollback function is run on all nodes and followed by unlocking and the transaction is aborted.

## Example
_(in pseudo code)_
```
# NewSimpleTxn returns a transaction object following the simple transaction template
NewSimpleTxn(nodes, lockobj, stage, commit, store, rollback) {
  lockStep = Step{Func: DefaultLock(lockobj), Undo: DefaulUnlock(lockobj), Nodes: [LEADER]}
  stageStep = Step{Func: stage, Undo: nil, Nodes: [ALL]}
  commitStep = Step{Func: commit, Undo: rollback, Nodes: [ALL]}
  storeStep = Step{Func: store, Undo: nil, Nodes: [LEADER]}
  unlockStep = Step{Func: DefaultUnlock(lockobj), Undo: nil, Nodes: [LEADER]}

  return Txn{Nodes:nodes, Steps:[lockStep, stageStep, commitStep, storeStep, unlockStep]}
}

# Running a simple transaction
txn = NewSimpleTxn(["node1","node2"], "some-volume", stagefunc, commitfunc, storefunc, rollbackfunc)
result, error = txn.Perform() # Perform will run the steps in order and return the result

# Free-form transaction
# Taking replace-brick as an example (this is not a 100% correct replace-brick transaction)
nodes = ["source", "dest"]
checkdest = Step{Func: CanBrickBeCreated, Undo: nil. Nodes:["dest"]}
updateVolume = Step{Func: UpdateVolumeWithNewInfo, Undo: RevertToOldVolume, Nodes[LEADER]}
stopsource = Step{Func:Stopbrick, Undo: StartBrick, Nodes:["source"]}
startdest = Step{Func:Startbrick, Undo: StopBrick, Nodes:["dest"]}
replaceBrick = Txn{Nodes: nodes, Steps:[checkdest, updateVolume, startdest, stopsource]}
res, err = replaceBrick.Perform()
```
