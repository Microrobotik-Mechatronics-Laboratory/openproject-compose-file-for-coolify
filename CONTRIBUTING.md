# Contributing

Thank you for your interest in contributing to this project. This guide explains how to report issues, suggest improvements, and submit changes.

## Reporting Issues

If you encounter a problem:

1. **Search existing issues** to check if it has already been reported.
2. **Open a new issue** using the appropriate template:
   - **Bug Report** -- for deployment failures, configuration errors, or unexpected behavior.
   - **Feature Request** -- for new features, improvements, or additional documentation.
3. **Include details:**
   - Coolify version (e.g. v4.0.0-beta.380)
   - OpenProject image tag (e.g. 17-slim)
   - Relevant logs (from Coolify's Logs tab)
   - Steps to reproduce the issue

## Suggesting Improvements

Ideas for improvement are welcome. Open an issue with the **Feature Request** template and describe:

- What problem you are trying to solve
- Your proposed solution
- Any alternatives you considered

## Submitting Changes

1. **Fork** the repository
2. **Create a branch** from `main`:
   ```bash
   git checkout -b feature/your-feature-name
   ```
3. **Make your changes** and test them on a Coolify instance if possible
4. **Commit** with a clear, descriptive message:
   ```bash
   git commit -m "Add SMTP configuration example for Gmail"
   ```
5. **Push** your branch and **open a Pull Request** against `main`

### Pull Request Guidelines

- Keep changes focused -- one feature or fix per PR
- Update documentation (README, .env.example) if your change affects configuration
- Test your docker-compose changes on a Coolify v4.x instance before submitting
- Use clear commit messages in English

## Code of Conduct

Be respectful and constructive. We are all here to make self-hosted project management easier.

## Questions

If you have questions about the project or deployment, open a **Discussion** or an issue. We are happy to help.
