# Building an AI Interview Coach with React, Cloudflare Workers, and LLM APIs

## Overview

The AI Interview Coach is a full-stack application designed to help users practice technical and behavioral interviews with an AI-powered feedback loop. Instead of simply generating interview questions, the system guides a user through a complete practice session: generating questions, collecting responses, analyzing those responses, and producing structured feedback.

I built the application using React for the frontend, Cloudflare Workers for the serverless backend, and the Groq API for large language model inference.

The main engineering challenge was not calling an LLM API. That part is relatively straightforward. The more interesting problem was designing the surrounding system so that model outputs were structured, predictable, useful, and easy for the frontend to consume.

This document explains the architecture, request flow, prompt design, error handling, and several lessons I learned while building the application.

---

## System Architecture

At a high level, the application contains three major components:

1. **React frontend**
2. **Cloudflare Workers backend**
3. **Groq LLM API**

The request flow looks like this:

```text
User
  |
  v
React Frontend
  |
  | HTTPS request
  v
Cloudflare Worker
  |
  | Prompt + structured input
  v
Groq API / LLM
  |
  | Generated response
  v
Cloudflare Worker
  |
  | Validation / formatting
  v
React Frontend
  |
  v
User Feedback
```

Keeping the model interaction behind a backend layer was an important architectural decision. The frontend never needs direct access to the model provider's API credentials, and the backend can control prompts, validate requests, normalize responses, and handle failures before data reaches the user interface.

---

## Frontend

The frontend is built with React and is responsible for the user-facing interview experience.

Its responsibilities include:

- collecting interview preferences,
- displaying generated questions,
- accepting user responses,
- sending responses to the backend,
- displaying evaluation results,
- managing interview state, and
- presenting errors when a request cannot be completed.

The frontend does not need to understand how the LLM works internally. It sends structured application data to an API endpoint and receives a predictable response.

For example, a simplified request might look like:

```json
{
  "role": "Software Engineer",
  "experienceLevel": "Intern",
  "questionType": "behavioral"
}
```

Instead of building prompts directly inside the React application, the frontend communicates intent through structured fields.

This separation makes the application easier to maintain because prompt engineering can evolve independently from the UI.

---

## Serverless Backend

I used Cloudflare Workers as the backend layer between the React application and the Groq API.

The Worker has several responsibilities:

- validating requests,
- constructing model prompts,
- calling the LLM API,
- handling API failures,
- processing model responses,
- returning standardized JSON to the frontend.

A simplified endpoint might follow this pattern:

```javascript
export default {
  async fetch(request, env) {
    if (request.method !== "POST") {
      return new Response("Method not allowed", {
        status: 405
      });
    }

    try {
      const body = await request.json();

      if (!body.role) {
        return Response.json(
          { error: "Role is required" },
          { status: 400 }
        );
      }

      const result = await generateInterviewQuestion(
        body,
        env.GROQ_API_KEY
      );

      return Response.json(result);
    } catch (error) {
      return Response.json(
        { error: "Unable to generate interview question" },
        { status: 500 }
      );
    }
  }
};
```

The important part is not the exact implementation, but the separation of responsibilities.

The frontend handles presentation.

The Worker handles application logic and model interaction.

The model handles language generation and evaluation.

---

## Designing Prompts for Predictable Output

One of the first issues I encountered was that natural-language model responses were difficult for the application to consume reliably.

For example, asking:

```text
Evaluate this interview response.
```

could produce very different formats:

```text
This was a strong answer overall...
```

or:

```text
Score: 8/10

Strengths:
- Good explanation
- Strong example
```

or even a long paragraph with no consistent structure.

That creates a problem for the frontend because it cannot reliably determine which part represents a score, feedback, strengths, or recommendations.

I improved this by explicitly defining the expected output structure.

For example:

```text
Evaluate the candidate's answer.

Return the result with:

score
strengths
weaknesses
suggestions

The score must be from 1 to 10.
```

Ideally, the backend requests or enforces JSON-compatible output:

```json
{
  "score": 8,
  "strengths": [
    "Clearly explained personal responsibility",
    "Included a measurable outcome"
  ],
  "weaknesses": [
    "Could provide more technical detail"
  ],
  "suggestions": [
    "Explain how the solution was implemented"
  ]
}
```

Structured responses significantly simplify frontend development.

Instead of trying to parse arbitrary text, React can render individual fields directly.

---

## Evaluating Interview Responses

Generating questions is only one part of the application.

The more interesting feature is evaluating a user's response.

A typical evaluation request contains:

- the interview question,
- the candidate's answer,
- the target role,
- optional experience level, and
- evaluation criteria.

For behavioral questions, the system can evaluate characteristics such as:

- clarity,
- relevance,
- ownership,
- specificity,
- use of measurable outcomes, and
- completeness of the example.

For technical questions, the criteria can change to emphasize:

- correctness,
- reasoning,
- tradeoffs,
- complexity,
- terminology, and
- communication.

This is an important distinction.

A single generic evaluation prompt is unlikely to work equally well for every type of interview question.

The evaluation criteria should reflect the task being evaluated.

---

## Handling LLM Output Carefully

One lesson I learned while building this project is that an LLM response should not automatically be treated as correct simply because it is formatted confidently.

The application needs defensive logic around the model.

For example, suppose the expected response contains:

```json
{
  "score": 7,
  "strengths": [],
  "weaknesses": [],
  "suggestions": []
}
```

The backend should verify assumptions before returning it.

Questions worth checking include:

- Is the score actually a number?
- Is it between 1 and 10?
- Are required fields present?
- Did the model return valid JSON?
- Did the API return an empty response?
- Was the request rejected or rate limited?
- Did the model return unrelated content?

A simplified validation function could look like:

```javascript
function validateEvaluation(result) {
  if (!result) {
    throw new Error("Empty model response");
  }

  if (
    typeof result.score !== "number" ||
    result.score < 1 ||
    result.score > 10
  ) {
    throw new Error("Invalid evaluation score");
  }

  if (!Array.isArray(result.strengths)) {
    throw new Error("Invalid strengths field");
  }

  if (!Array.isArray(result.suggestions)) {
    throw new Error("Invalid suggestions field");
  }

  return result;
}
```

This type of validation is simple, but it prevents malformed model outputs from silently becoming application state.

---

## Error Handling

AI applications introduce failure modes that normal deterministic APIs may not.

A request can fail because of:

- network problems,
- provider downtime,
- rate limits,
- malformed model output,
- invalid application input,
- token limits, or
- unexpected model behavior.

Instead of exposing provider-specific errors directly to the user, the backend translates them into application-level responses.

For example:

```json
{
  "error": "We couldn't evaluate this response. Please try again."
}
```

At the same time, developers need enough internal information to debug the problem.

This creates two different error audiences:

### User-facing errors

These should be simple and actionable.

```text
We couldn't generate feedback. Please try again.
```

### Developer-facing errors

These should include enough context for debugging.

```text
Groq API request failed:
status=429
endpoint=/evaluate
```

Separating these concerns improves both usability and maintainability.

---

## Protecting API Credentials

Another reason I used a backend layer was to prevent API credentials from being exposed in browser code.

A client-side implementation like:

```javascript
const apiKey = "my-secret-key";
```

would expose the credential to anyone using the application.

Instead, credentials should be stored as environment variables or platform secrets.

For example:

```javascript
env.GROQ_API_KEY
```

The browser calls the Cloudflare Worker, and only the Worker communicates with the model provider.

The flow becomes:

```text
Browser
   |
   | No model API key
   v
Cloudflare Worker
   |
   | Secure environment variable
   v
Groq API
```

This is an important pattern for almost any application integrating third-party APIs.

---

## Why I Used a Serverless Backend

Cloudflare Workers were a good fit because the backend primarily performs lightweight request processing and external API calls.

The architecture provides several benefits:

- no traditional server management,
- straightforward deployment,
- environment-based secret management,
- low-latency API handling,
- easy integration with a web frontend.

For a project like this, a traditional always-running server would have added infrastructure complexity without providing a clear benefit.

However, a larger version of the application might eventually require additional services for persistent data, authentication, analytics, or long-running processing.

---

## What I Would Improve Next

If I continued developing the system, I would focus on several areas.

### 1. Stronger evaluation

LLM evaluation can itself be inconsistent.

I would build a fixed evaluation dataset containing example interview responses and expected scoring characteristics. This would make it possible to compare prompt versions systematically rather than relying on subjective manual testing.

### 2. Automated prompt testing

Prompt changes can improve one scenario while unintentionally making another worse.

I would create regression tests that run representative inputs whenever important prompts change.

For example:

```text
Prompt Version A
        |
        v
Evaluation Dataset
        |
        v
Scores / Structured Outputs
        |
        v
Compare Against Version B
```

This would make prompt development more similar to traditional software testing.

### 3. Observability

I would add metrics for:

- request latency,
- model failures,
- invalid structured responses,
- token usage,
- evaluation completion rate, and
- user retries.

These metrics would help distinguish model-quality problems from infrastructure problems.

### 4. Persistent user sessions

A database could store previous interview sessions, allowing users to track improvement over time.

The system could then identify trends such as:

```text
Behavioral clarity:
6.2 -> 7.1 -> 8.0

Technical explanation:
5.8 -> 6.4 -> 7.3
```

This would transform the product from a single-session tool into a longer-term coaching platform.

### 5. Retrieval-Augmented Generation

A future version could use a user's resume or a specific job description as context.

Instead of generating generic questions, the system could retrieve relevant information and generate questions grounded in the candidate's actual experience and target role.

---

## Key Lessons

Building the AI Interview Coach changed how I think about developing applications with large language models.

The most important lesson was that integrating an LLM is only a small part of building an AI product.

The surrounding engineering matters just as much:

- defining clear interfaces,
- structuring model outputs,
- validating responses,
- protecting credentials,
- handling failures,
- testing behavior,
- monitoring performance, and
- designing around the user's actual goal.

A model may provide intelligence, but the application around it determines whether that intelligence becomes a reliable product.

That distinction is what makes AI application engineering especially interesting to me.
