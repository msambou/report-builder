Prior Knowledge:
- Students already know Poetry

Demo: 
Using the solution from the recitation on P0,
- Start by adding dependencies - pydantic, pytest, etc
- Refactor the code to use FastAPI
- Explain what Pydantic models are and how they’re helpful for validating user requests
- Add a validator for a particular input parameter: for eg. Age
    - If age > 18 then do….
    - You understand the recitation best, so do what’s best in your opinion
- Create unit tests
- Walk them through how unit tests are added
- Mock API responses. Remember in the project, there’s a part where they had to mock the LLM service response.
- How to ruff for formatting

NB:
Don’t bother about the CI/CD part. That’s fairly easy
I want you to rather direct your efforts into refactoring, Pydantic models, testing as these are the areas I foresee them to struggle. They already have a good foundation of GitHub Actions.

Kahoot
The Kahoot should cover the following areas:
- A question about the need for refactoring
- What decorator do you use when writing a validator function?
- A question about mocking an API response. 
- A question about the difference between unit and integration tests
- A question about formatting tools, eg. Ruff