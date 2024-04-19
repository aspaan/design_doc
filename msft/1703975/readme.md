# Design Document for CI/CD Test Integration

1. **[Introduction](#introduction)**
   - [Assumptions](#assumptions)

2. **[Solution Approaches](#solution-approaches)**
   - [Approach 1: Static Partitioning with Dynamic Execution](#approach-1-static-partitioning-with-dynamic-execution)
     - [Description](#description)
     - [Implementation Outline](#implementation-outline)
     - [Pros](#pros)
     - [Cons](#cons)
   - [Approach 2: Dynamic Partitioning and Load Balancing](#approach-2-dynamic-partitioning-and-load-balancing)
     - [Description](#description-1)
     - [Implementation Outline](#implementation-outline-1)
     - [Pros](#pros-1)
     - [Cons](#cons-1)

3. **[Recommendation](#recommendation)**

4. **[Approach 2: Implementation](#approach-2-implementation)**
   - [YAML Configuration for GitHub Actions](#yaml-configuration-for-github-actions)
   - [TypeScript Code (`runTests.js`)](#typescript-code-runtestsjs)
   - [Explanation](#explanation)


## Introduction
This design document details two distinct solutions to integrate a suite of 3,000 end-to-end Playwright tests into the pull request validation pipeline for a software development project. The goal is to minimize the total execution time of the validation pipeline while adhering to a specified duration limit through efficient parallel execution across multiple agents.

### Assumptions
- The CI/CD system can be any platform that supports parallel execution (Azure DevOps, GitHub Actions).
- The tests are not uniformly distributed in terms of execution time.
- The external test selection API is reliable and returns an accurate subset of tests along with their estimated execution times.
- Sufficient agents are available to support parallel execution.

## Solution Approaches

### Approach 1: Static Partitioning with Dynamic Execution
**Description:**
This approach involves statically dividing the tests into predefined shards, but dynamically assigning these shards to agents based on the current load and priority. The partitioning could be performed in a way that attempts to balance the total estimated execution time across each shard, using historical data or estimations provided by the test selection API.

![Sequence diagram](https://diagrams.helpful.dev/d/d:5WgwB0fo)

**Implementation Outline:**
1. **YAML Configuration:** Define multiple jobs in the CI/CD pipeline, each responsible for a specific shard of tests. These jobs are triggered concurrently.
2. **Code Execution:** A TypeScript script fetches the list of relevant tests from the API and dynamically dispatches them to the predefined shards. Agents pick up jobs as they become available, optimizing resource usage.

**Pros:**
- Simplifies management by maintaining a fixed number of shards.
- Efficient utilization of resources by dynamic dispatch based on agent availability.

**Cons:**
- Less flexibility in balancing load dynamically across the agents.
- Potential for idle times if shards are not perfectly balanced.

### Approach 2: Dynamic Partitioning and Load Balancing
**Description:**
This solution leverages a more dynamic strategy where each agent fetches tests directly from a queue. The queue is populated based on the subset of tests deemed relevant by the test selection API, ensuring that the workload is balanced in real-time across available agents.

Here is the sequence diagram illustrating the dynamic partitioning and load balancing approach for CI/CD pipelines:

![Sequence diagram](https://diagrams.helpful.dev/d/d:NzDVafwU)

**Implementation Outline:**
1. **YAML Configuration:** Configure a single job that initializes multiple parallel agents. Each agent independently requests a batch of tests from a central queue until all tests are assigned.
2. **Code Execution:** A TypeScript script acts as a coordinator. It retrieves the list of tests from the API, calculates an optimal distribution based on test duration, and feeds these into a queue. Agents pull from this queue dynamically.

**Pros:**
- High flexibility and adaptability to varying loads and test durations.
- Potential for very efficient resource utilization, minimizing idle time.

**Cons:**
- Increased complexity in managing a dynamic queue and ensuring fair distribution among agents.
- Potential overhead from agents frequently querying for new tests.

## Recommendation
**Approach 2: Dynamic Partitioning and Load Balancing** is recommended due to its superior flexibility and efficiency in resource utilization. This method ensures that all agents are consistently occupied and can adapt to varying test durations and agent availability better than static partitioning.

This approach also aligns better with the goal of minimizing pipeline execution time while adhering to the predetermined duration constraints. It might require more sophisticated setup and management, but the benefits in terms of scalability and efficient resource usage are significant, especially as the number of tests or agents grows.

### Approach 2: Implementation

#### YAML Configuration for GitHub Actions
```yaml
name: Dynamic Test Distribution CI Pipeline

on:
  pull_request:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node: [1, 2, 3, 4] # Define the number of parallel agents dynamically based on need
    steps:
      - uses: actions/checkout@v2
      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '14'
      - name: Install dependencies
        run: npm install
      - name: Run tests dynamically
        run: node ./scripts/runTests.js
        env:
          MATRIX_NODE: ${{ matrix.node }}
```

#### TypeScript Code (`runTests.js`)
```typescript
const fetch = require('node-fetch');
const TOTAL_AGENTS = 4; // This should match the matrix size in the YAML

async function fetchTests(agentIndex: number) {
    // Example POST request to fetch relevant tests
    const response = await fetch('https://tests-selection.azurewebsites.net/api', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(["changed_file1.ts", "changed_file2.ts"]) // Ideally, this would be dynamic based on the PR
    });
    const data = await response.json();

    return data.filter((_, index) => index % TOTAL_AGENTS === agentIndex);
}

async function runTests() {
    const agentIndex = parseInt(process.env.MATRIX_NODE) - 1;
    const testsToRun = await fetchTests(agentIndex);

    console.log(`Agent ${agentIndex + 1} running tests:`, testsToRun.map(test => test.test));
    // Here you would add logic to execute the tests, e.g., using Playwright
}

runTests().catch(err => console.error(err));
```

### Explanation
**YAML Configuration:**
- Defines a GitHub Actions workflow that triggers on pull requests to the `main` branch.
- Sets up a job to run tests on `ubuntu-latest`.
- Uses a matrix strategy to create four parallel agents.
- Each agent runs a Node.js script to fetch and execute a portion of the tests dynamically.

**TypeScript Code (`runTests.js`):**
- Implements an asynchronous function to fetch the relevant tests for each agent from the API.
- Filters tests so each agent processes a unique subset, thereby distributing the workload.
- Placeholder for test execution, which would involve integrating with a test runner like Playwright.

This setup ensures that the tests are distributed dynamically among the available agents, allowing for flexible scaling and efficient pipeline execution.