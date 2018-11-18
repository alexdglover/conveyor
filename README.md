# conveyor
Pipelines are for shit. Conveyors move product. This is a CI/CD platform written in Python


# Services

Conveyor is meant to be a distributed build system. It is comprised of two 
services - Orchestrators and Builders

## Orchestrator

The Orchestrator is the core of Conveyor, and serves multiple purposes:

* web UI for human interaction
* API backend for progammatic interaction
* CRUD operations for jobs

A minimum of one Orchestrator is required, but multiple Orchestrators can be 
deployed for high availability if using an external database and queue.

## Builder

The Builder is what actually does the work. Builders are dumb, can be hosted 
anywhere (as long as they can reach the database and queue), and are stateless. 

# Logical Objects

## Jobs

Conveyor's primary focus is to provide interoperable, modular jobs. Jobs can 
execute any set of arbitrary commands within the job, but they have 'typed' 
inputs and outputs. This allows for artifacts or state to be passed from one 
job to the next, decision-based workflows, sane retries, and more. This is the 
main differentiator for Conveyor.

## Projects

Projects can be defined within an Orchestrator, or can be defined 
programmatically in the source or repository.
