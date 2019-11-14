---
title: "Use the default Evaluator"
linkTitle: "Use the default Evaluator"
weight: 3
description: >
  A tutorial on how to use the default Evaluator to resolve Match proposal conflicts.
---

## Objectives

- Understands how and why we need the default Evaluator to resolve proposal conflicts.
- Instructs your MatchFunction to use the default Evaluator.

## Prerequisites

### Basic Matchmaker tutorial

It is highly recommended that you run the [Basic Matchmaker tutorial]({{< relref "../Matchmaker101/_index.md" >}}) as it introduces the core concepts that this tutorial builds upon. After running the basic tutorial, please run the following command to delete its namespace before proceeding:

```bash
kubectl delete namespace mm101-tutorial
```

### Set up Image Registry

Please setup an Image registry(such as [Docker Hub](https://hub.docker.com/) or [GC Container Registry](https://cloud.google.com/container-registry/)) to store the Docker Images that will be generated in this tutorial. Once you have set this up, here are the instructions to set up a shell variable that points to your registry:

```bash
REGISTRY=[YOUR_REGISTRY_URL]
```

If using GKE, you can populate the image registry using the command below:

```bash
REGISTRY=gcr.io/$(gcloud config list --format 'value(core.project)')
```

### Get the tutorial template

Make a local copy of the [tutorials Folder](https://github.com/googleforgames/open-match/blob/{{< param release_branch >}}/tutorials/default_evaluator). Use `tutorials/default_evaluator` as a working copy for all the instructions in this tutorial.

For convenience, set the following variable:

```bash
TUTORIALROOT=[SRCROOT]/tutorials/default_evaluator
```

### Create the tutorial namespace

Run this command to create a namespace default-eval-tutorial in which all the components for this tutorial will be deployed.

```bash
kubectl create namespace default-eval-tutorial
```

### Reference Reading

It is highly recommended that you read the [Evaluator Guide]({{< relref "../../Guides/evaluator.md" >}}) to familiarize yourself with the lifecycle of a Match proposal through the synchronization and evaluation phases. Also, keep the [API Reference]({{< relref "../../reference/api.md" >}}) handy to look up Open Match specific terminology used in this document.

{{% alert title="Note" color="info" %}}
A complete [solution](https://github.com/googleforgames/open-match/blob/{{< param release_branch >}}/tutorials/default_evaluator/solution) for this tutorial is in the folder `tutorials/default_evaluator/solution`.
{{% /alert %}}

## Evaluation

### Overview

In the [basic Matchmaker]({{< relref "../Matchmaker101/_index.md" >}}), each Ticket had only one game-mode and the Director generated a MatchProfile per game-mode. Thus concurrently executing MatchFunction (for each MatchProfile) worked with completely partitioned (by game-mode) Ticket Pools. However, this may not always be the case. There are scenarios in which, the Pools specified in concurrent MatchProfiles may have an overlap and thus concurrently executing MatchFunctions could use overlapping Ticket in its Proposals.

In a lot of real matchmaking scenarios, this overlap is desired. To generate high-quality Matches, you may want your Ticket to be considered concurrently for multiple possible MatchProfiles and then be able to compare the quality of these overlapping Matches, picking the highest quality Match and discarding the rest. The ability to provide this model of executing concurrent MatchFunctions on overlapping Ticket Pool is a core value proposition of Open Match.

To allow generation of proposals using overlapping Tickets but still provide unique Matches to the Director, Open Match synchronizes all the proposals generated by concurrently executing MatchFunctions and passes these to an Evaluator to perform de-collision.

Since basic cases may either not need an evaluation or may need very simple evaluation, Open Match provides a default Evaluator as an optional component that may work in most scenarios. At a high level, there are three evaluation scenarios for you to consider when using Open Match:

1. Your MatchProfiles completely partition the Ticket Pool and so you will never have collisions(the basic Matchmaker falls under this usage). The default Evaluator handles this scenario and you do not have to make any changes.
2. Your MatchProfiles may use overlapping Ticket Pools - but each MatchFunction can simply generate a quality score for that Match based on certain game-specific criteria (latency, MMR, etc.). The default Evaluator can be used for this scenario by following instructions in this tutorial to add specific information needed for evaluation to the Match in your MatchFunction.
3. You have complex evaluation logic that cannot be simplified to a score for a Match but rather involves comparing some details of the overlapping Matches. Open Match provides you with the ability to plug in your custom Evaluator to handle this case. See the [tutorial for using a Custom Evaluator]({{< relref "../CustomEvaluator/_index.md" >}}) for details.

### Default Evaluator

The Evaluator for Open Match at a high level accepts Match proposals (that may be overlapping) and returns non-overlapping result Matches. Open Match provides an optional default score based Evaluator that you can deploy with core Open Match installation.

The default Evaluator expects that the MatchFunction populate a 'score' for the generated Matches. When it is invoked with a list of proposals, it simply sorts the Matches by score and in case of collision, picks the Match with the highest score as a result and discards the other colliding matches.

The MatchFunction may use a game-specific information (MMR, latency, etc.) to compute the score for this Match. It populates the score in a DefaultEvaluationCriteria proto that is shown below.

```proto
message DefaultEvaluationCriteria {
  double score = 1;
}
```

The MatchFunction embeds the populated DefaultEvaluationCriteria in the Match's Extensions field (Extensions are a mechanism Open Match uses to pass blobs of specific information across components that extend Open Match). The default Evaluator accesses the score for the Match from this field and uses it as a proxy for the Match quality. This tutorial provides step-by-step instructions and samples to generate the score and populate the DefaultEvaluationCriteria for a Match.

## Using the Default Evaluator

### Scenario Overview

For this tutorial, we will tweak the basic matchmaking scenario to enable a Player to select more than one game-mode to be considered for finding a Match. Given that the Director for our Matchmaker generates a MatchProfile per game-mode, concurrent MatchFunction executions handling different MatchProfiles will both consider the same Ticket(Player) for their respective Match proposals. We will use the default Evaluator provided by Open Match to resolve conflicts in this scenario.

To generate a score (for evaluation) indicating the Match quality, we will introduce a new property on each Ticket indicating the time at which the Ticket entered matchmaking queue. The MatchFunction can then aggregate the wait times for all the Tickets in a Match to generate a score for that Match. Higher the score, longer the total wait times and higher the likelihood that the default Evaluator will pick the Match in case of a conflict.

Here are the changes we need to make at a high level to our components to achieve this:

- The Game Frontend will generate Tickets with two game-mode preferences.
- The Game Frontend will add a new SearchField indicating the time of entry in matchmaking queue.
- The MatchFunction will generate the score for each Match to be used as evaluation criteria.

The below sections provide details of these changes:

### Game Frontend

This tutorial provides a Game Frontend scaffold (`$TUTORIALROOT/frontend`) that generates a Ticket with the game-mode specified (as implemented for the basic Matchmaker). The mock Ticket generation is implemented in `makeTicket()` in `$TUTORIALROOT/frontend/ticket.go`.

The below snippet adds a new SearchField representing the time of entry in matchmaking queue:

```golang
func makeTicket() *pb.Ticket {
  ticket := &pb.Ticket{
    SearchFields: &pb.SearchFields{
      Tags: gameModes(),
      DoubleArgs: map[string]float64{
        "time.enterqueue": enterQueueTime(),
      },
    },
  }

  return ticket
}

func enterQueueTime() float64 {
  // Implement your logic to return a random time interval.
}

func gameModes() []string {
  modes := []string{"mode.demo", "mode.ctf", "mode.battleroyale", "mode.2v2"}
  // Implement your logic to return any two of the above game-modes randomly
}
```

Please update the Ticket creation logic in `$TUTORIALROOT/frontend/ticket.go` with the above snippet adding your logic to pick the random time and game-modes.

### MatchFunction

This tutorial provides a MatchFunction scaffold (`$TUTORIALROOT/matchfunction`) that generates Match proposals for the MatchProfile that gets passed in. The core matchmaking logic is implemented in `makeMatches()` in `$TUTORIALROOT/matchfunction/mmf/matchfunction.go`.

Just because a Ticket has multiple game-modes, the MatchFunction will now consider Tickets for Match generation that other concurrent proposals use. Thus without any change to the core matchmaking logic here, we will generate overlapping results across MatchFunction executions.

To use default Evaluator, the MatchFunction needs to calculate a score for each Match and populate that in the DefaultEvaluationCriteria field. The code below demonstrates how to populate this field in the MatchFunction:

```golang

func makeMatches(p *pb.MatchProfile, poolTickets map[string][]*pb.Ticket) ([]*pb.Match, error) {
...
// Add score calculation logic here.
    matchScore := scoreCalculator(matchTickets)
    evaluationInput, err := ptypes.MarshalAny(&pb.DefaultEvaluationCriteria{
      Score: matchScore,
    })

    if err != nil {
      log.Printf("Failed to marshal DefaultEvaluationCriteria, got %w.", err)
      return nil, fmt.Errorf("Failed to marshal DefaultEvaluationCriteria, got %w", err)
    }

  ...
}

func scoreCalculator(tickets []*pb.Ticket) float64 {
  matchScore := 0.0
  // Add your logic to compute the score for this Match here
  return matchScore
}
```

Add the above snippet to the MatchFunction and populate the logic for score calculation. One strategy you may use is to aggregate the wait time for each Ticket and use that value as the score.

## Build, Deploy, Run

### Build and Push Container images

Now that you have customized these components, please run the below commands from `$TUTORIALROOT` to build new images and push them to your configured image registry.

```bash
docker build -t $REGISTRY/default-eval-tutorial-frontend frontend/.
docker push $REGISTRY/default-eval-tutorial-frontend
docker build -t $REGISTRY/default-eval-tutorial-director director/.
docker push $REGISTRY/default-eval-tutorial-director
docker build -t $REGISTRY/default-eval-tutorial-matchfunction matchfunction/.
docker push $REGISTRY/default-eval-tutorial-matchfunction
```

### Deploy and Run

Run the below command in the `$TUTORIALROOT` path to deploy the MatchFunction, Game Frontend and the Director to the `default-eval-tutorial` namespace:

```bash
sed "s|REGISTRY_PLACEHOLDER|$REGISTRY|g" matchmaker.yaml | kubectl apply -f -
```

### Output

All the components in this tutorial simply log their progress to stdout. Thus to see the progress, run the below commands:

```bash
kubectl logs -n default-eval-tutorial pod/default-eval-tutorial-frontend
kubectl logs -n default-eval-tutorial pod/default-eval-tutorial-director
kubectl logs -n default-eval-tutorial pod/default-eval-tutorial-matchfunction
```

To check the logs from the default Evaluator, run the following commands:

```bash
// Locate the Evaluator pod(s) using the following command
kubectl get pods -n open-match

// Check the logs from the Evaluator pod
kubectl logs -n open-match <Evaluator Pod>
```

## Cleanup

Run the below command to remove all the components of this tutorial:

```bash
kubectl delete namespace default-eval-tutorial
```

{{% alert title="Note" color="info" %}}
This will still keep the Open Match core running in `open-match` namespace for reuse by the other exercises.
{{% /alert %}}
## What Next

There are scenarios where evaluating overlapping Matches may need context from both Matches and thus the logic may not be represented by a single score computed in isolation for a Match. In such cases, you can [build your custom Evaluator]({{< relref "../CustomEvaluator/_index.md" >}}) and connect that to Open Match.