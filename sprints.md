# 🏁 Challenge Execution Plan (Sprints)

## 🏃 Sprint 1: Setup, Database, and Seed

**Goal:** Initialize the environment, set up the SQLite database, and seed the initial data.

**Tasks for the Agent:**

- Configure Prisma with SQLite and create the following schema models: `User`, `Quiz`, `Question`, `Option`, `Attempt`, `AttemptAnswer` (to store individual user submissions), and `Notification`.
- Apply these schema rules: `Attempt` relates to `User` and `Quiz`. `AttemptAnswer` relates to `Attempt`, `Question`, and `Option`. `Notification` has a 1-1 relationship with `Attempt`.
- Create a global `PrismaModule` and `PrismaService` extending PrismaClient and implementing `OnModuleInit`. Export it so other modules can use it.
- Create a `seed.ts` script to populate the database with 2 quizzes about AI concepts (at least 5 questions each, including correct answers and explanations) and 1 default user (hardcode the user ID as 'user-123' to easily test later).
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

## 🏃 Sprint 3: The Core (Start Attempt, Submission & Scoring Logic)

**Goal:** Allow users to start an attempt, process their submissions, store individual answers, calculate scores, and trigger the async completion event.

**Tasks for the Agent:**

- Generate the `AttemptModule`, `AttemptController`, and `AttemptService`.
- Create the **`POST /attempts/start`** route:
  - Payload: `userId` and `quizId`.
  - Logic: Create a new `Attempt` in the database with a 'IN_PROGRESS' status. Return the `Attempt` ID along with the Quiz questions and options. **CRITICAL:** Do NOT leak `isCorrect` or `explanation` here.
- Create the **`POST /attempts/:attemptId/submit`** route:
  - Payload: An `answers` array `[{questionId, optionId}]`.
  - Implement the following business logic in the Service:
    1. Fetch the in-progress `Attempt` and the quiz answer key from the database.
    2. Save each answer into the database using the `AttemptAnswer` model (satisfying the 'individual answer submissions per attempt' requirement).
    3. Calculate the total score and percentage.
    4. Generate a performance `message` based on the percentage (e.g., <50% try again, >80% excellent).
    5. Update the `Attempt` status to 'COMPLETED' with the final score, and create a `Notification` record with status 'PENDING'.
    6. Return the score, message, and a detailed question breakdown to the user (now it is safe to expose whether they got it right, the correct option, and the explanation).
    7. Emit a `quiz.completed` event passing the `attemptId` using `@nestjs/event-emitter`.

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
