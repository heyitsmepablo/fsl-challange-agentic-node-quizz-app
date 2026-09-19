# 🏁 Challenge Execution Plan (Sprints)

## 🏃 Sprint 1: Setup, Database, and Seed

**Goal:** Initialize the environment, set up the SQLite database, and seed the initial data.

**Tasks for the Agent:**

- Initialize a basic NestJS project.
- Install Prisma, `@nestjs/swagger`, and `@nestjs/event-emitter`.
- Configure Prisma with SQLite and create the following schema models: `User`, `Quiz`, `Question`, `Option`, `Attempt`, and `Notification`.
- Apply these schema rules: `Attempt` relates to `User` and `Quiz`. `Notification` has a 1-1 relationship with `Attempt`.
- Create a `seed.ts` script to populate the database with 2 quizzes about AI concepts (at least 5 questions each, including correct answers and explanations) and 1 default user.
- Configure Swagger documentation in `main.ts`.

---

## 🏃 Sprint 2: Quizzes Module (The Security Rule)

**Goal:** Create the quiz reading and creation endpoints without leaking the answer key.

**Tasks for the Agent:**

- Generate the `QuizModule`, `QuizController`, and `QuizService`.
- Create the following routes:
  - `GET /quizzes` (list all quizzes)
  - `GET /quizzes/:id` (quiz details including questions and options)
  - `POST /quizzes` (create a new quiz)
- **CRITICAL CONSTRAINT:** In the GET endpoints, the Prisma query MUST use `select` to explicitly OMIT the `isCorrect` and `explanation` fields from options and questions. They must NEVER be returned in the JSON response payload.

---

## 🏃 Sprint 3: The Core (Submission & Scoring Logic)

**Goal:** Process user submissions, calculate scores, and trigger the async completion event.

**Tasks for the Agent:**

- Generate the `AttemptModule` and create the `POST /attempts/submit` route.
- The payload must accept: `userId`, `quizId`, and an `answers` array `[{questionId, optionId}]`.
- Implement the following business logic in the Service:
  1. Fetch the quiz answer key from the database (this query is allowed to read `isCorrect`).
  2. Calculate the total score and percentage.
  3. Generate a performance `message` based on the percentage (e.g., <50% try again, >80% excellent).
  4. Save the `Attempt` to the database and create a `Notification` record with status 'PENDING'.
  5. Return the score, message, and a detailed question breakdown to the user (now it is safe to expose whether they got it right, the correct option, and the explanation).
  6. Emit a `quiz.completed` event passing the `attemptId` using `@nestjs/event-emitter`.

---

## 🏃 Sprint 4: Asynchronism and Resilience

**Goal:** Handle the mock email notification without blocking the main thread, and build the user progress route.

**Tasks for the Agent:**

- Create the `NotificationModule` and a `NotificationService`.
- Create an event listener for `@OnEvent('quiz.completed', { async: true })`.
- Implement the listener logic:
  1. Fetch the `Attempt` (including `User` and `Quiz` data).
  2. Simulate sending an email by logging to the console (Name, Email, Quiz Title, Score, Percentage, Message, Date).
  3. Add `if (Math.random() > 0.8) throw new Error('Mock email failure');` to simulate a 20% failure rate.
  4. If successful, update the `Notification` status to 'SENT'. If an error is caught, update it to 'FAILED'.
- Create the `GET /users/:userId/progress` route in the User or Attempt module to list the user's attempt history (ensure retakes don't overwrite previous attempts) and calculate their overall average score.

---

## 🏃 Sprint 5: Tests and Review

**Goal:** Write unit tests to cover the mandatory challenge criteria.

**Tasks for the Agent:**

- Create unit tests using Vitest covering the following scenarios:
  1. The logic/function that calculates the Score.
  2. The logic/function that generates the Performance Message.
  3. The asynchronous workflow behavior (mock the EventEmitter and Prisma to guarantee that an exception in the notification changes the database status to FAILED but does not crash the submission process).
