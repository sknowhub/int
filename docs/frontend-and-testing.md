# Frontend & Testing Interview Guide

## 1. Overview
At Staff+ level, you are expected to understand modern frontend architecture and how quality is built into the system through comprehensive testing strategies. These two disciplines deeply intersect: a testable frontend requires clean architecture, and a reliable system demands rigorous testing at all levels. Interviewers will probe:

- **Frontend** – component design, state management, performance, accessibility, security, build tooling, micro-frontends, and architectural trade-offs.
- **Testing** – test pyramid, automation, unit/integration/e2e/contract testing, mocking strategies, TDD/BDD, performance testing, security testing, observability in testing, CI integration, and quality gates.

Real-world systems: large-scale SPAs, multi-platform applications, design systems, internal tooling, customer-facing dashboards, and the entire testing culture that supports continuous delivery.

## 2. Deep Knowledge Structure

### 2.1 Frontend Architecture

#### Core Concepts & Technologies
- **SPA vs MPA vs SSG/SSR**: React/Angular/Vue SPA, Next.js/Nuxt SSR, static site generation, hydration; tradeoffs: SEO, initial load, interactivity, server load.
- **Component Model**: reusable, composable; props/state/events; smart vs dumb components; component libraries (Storybook).
- **State Management**: local component state, lifted state, global stores (Redux, Zustand, Jotai, Vuex, NgRx), server state caching (React Query, SWR); when to use which.
- **Styling**: CSS Modules, CSS-in-JS, utility-first (Tailwind), design tokens; scalability and theming.
- **Routing**: client-side (React Router), nested routes, guarded routes; code-splitting/lazy loading.

#### Performance
- **Core Web Vitals**: LCP, FID/INP, CLS; how to measure and improve.
- **Optimization Techniques**: code splitting, tree shaking, image optimization (WebP, srcset), font loading, resource hints (preload, prefetch), CDN, caching (service workers, HTTP caching).
- **Bundle Size**: monitor with Webpack Bundle Analyzer, dynamic imports, module federation.
- **Rendering Performance**: virtual DOM diffing, reconciliation, avoid unnecessary re-renders (React.memo, useMemo, useCallback), change detection (Angular zone.js vs OnPush).
- **Network**: HTTP/2 multiplexing, GraphQL query optimization, lazy loading data.

#### Security
- **OWASP Top 10 for Frontend**: XSS (reflected/DOM), CSRF, clickjacking, insecure storage (localStorage), sensitive data exposure.
- **Content Security Policy (CSP)**, X-Content-Type-Options, X-Frame-Options.
- **Authentication and session management**: JWT stored in httpOnly cookie vs memory; token refresh.

#### Accessibility (a11y)
- **WCAG**: semantic HTML, ARIA roles, keyboard navigation, color contrast, screen reader support.
- **Testing**: axe-core, Lighthouse, manual testing.

#### Micro-Frontends
- **Architectural approaches**: iframe (bad), Webpack Module Federation, single-spa, qiankun, runtime integration via custom elements.
- **Trade-offs**: independent deployments vs. overhead, shared dependencies, CSS isolation, inter-app communication.
- **When to use**: scaling teams, independent features.

#### Tooling & Build
- **Bundlers**: Webpack, Vite, esbuild, Turbopack; config for production/development.
- **Linting/Formatting**: ESLint, Prettier; enforce code consistency.
- **TypeScript**: type safety, refactoring support, better developer experience.
- **CI/CD for Frontend**: static analysis, test, build, deploy to CDN; preview environments per PR.

### 2.2 Testing Discipline

#### Testing Pyramid (Frontend & Backend)
- **Unit Tests**: test individual functions, components (shallow/mount). Fast, reliable. Tools: Jest, Vitest.
- **Integration Tests**: test interactions between modules, API calls (mocked server). Tools: React Testing Library, MSW (Mock Service Worker).
- **E2E Tests**: full user workflows, browser automation. Tools: Cypress, Playwright, Selenium.
- **Snapshot Tests**: limited use, verify UI doesn’t change unexpectedly.

#### Testing Strategies
- **TDD**: write test first (red-green-refactor); enforces testability.
- **BDD**: behavior-driven; use Gherkin (Cucumber) for collaboration.
- **Property-based Testing**: generate random inputs (fast-check) to find edge cases.
- **Visual Regression Testing**: Percy, Applitools, Chromatic.
- **Accessibility Testing**: automated checks in axe-core as part of CI.
- **API Testing**: contract testing (Pact), component testing (Spring Boot test slices, Karate).
- **Performance Testing**: Lighthouse CI, k6 for API load/soak/stress tests; set performance budgets.

#### Mocking & Test Doubles
- **Stubs vs Mocks vs Spies vs Fakes**: Martin Fowler’s taxonomy; when to use what.
- **Mock Service Worker (MSW)**: intercept network requests at service worker level, ideal for React.
- **Testcontainers**: for integration tests with real databases, message queues.

#### Test Quality & Culture
- **Test Flakiness**: root cause analysis, quarantine flaky tests, fix or retire.
- **Code Coverage**: line/branch/function; 80% is a myth – focus on critical paths.
- **Mutation Testing**: Pitest for Java, Stryker for JS; measure test effectiveness.
- **Test Maintainability**: DRY test helpers, but readability > cleverness.
- **Shift-Left Testing**: testing early in lifecycle; developer-owned tests.

#### CI Integration & Quality Gates
- **Pre-push hooks**: lint, unit tests.
- **CI pipeline**: fast parallelized tests; fail fast.
- **Quality Gates**: code coverage thresholds, no critical vulnerabilities, SonarQube quality gate, performance budget.

## 3. Code & Examples

```javascript
// React component with state management and async data
import { useQuery } from '@tanstack/react-query';
import { fetchUser } from './api';

function UserProfile({ userId }) {
  const { data, isLoading, error } = useQuery(['user', userId], () => fetchUser(userId));

  if (isLoading) return <Spinner />;
  if (error) return <Error message="Failed to load user" />;
  return <div>{data.name}</div>;
}

// Unit test with React Testing Library and MSW
import { render, screen, waitFor } from '@testing-library/react';
import { rest } from 'msw';
import { setupServer } from 'msw/node';
import UserProfile from './UserProfile';

const server = setupServer(
  rest.get('/api/users/:id', (req, res, ctx) => {
    return res(ctx.json({ id: '1', name: 'John' }));
  })
);
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('loads and displays user', async () => {
  render(<UserProfile userId="1" />);
  await waitFor(() => screen.getByText('John'));
});
```

```yaml
# GitHub Actions CI for frontend with lint, test, and performance budget
name: Frontend CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4 with { node-version: 20, cache: 'npm' }
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage
      - run: npm run build
      - name: Check bundle size
        run: npx bundlesize
      - name: Lighthouse audit
        run: npx lighthouse-ci --performance=90 --accessibility=100
```

```java
// Spring Boot controller test slice example (Testing)
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mvc;
    @MockBean UserService userService;

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.getUser(1L)).thenReturn(new User(1L, "John"));
        mvc.perform(get("/users/1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.name").value("John"));
    }
}
```

## 4. Interview Intelligence

### Topic: Micro-Frontends Decision

#### ❌ Mid-level Answer
“We can split the UI into separate apps, each deployed independently. Use iframes or module federation.”

#### ✅ Senior Answer
“For a large app with multiple teams, micro-frontends help scale development. We used Module Federation in Webpack 5 to compose a shell app that loads remote micro-apps at runtime. We ensured shared dependencies to avoid duplication and CSS isolation via CSS Modules. Inter-app communication via custom events. Trade-off: increased complexity in CI and orchestration but each team can release independently.”

#### 🚀 Staff-level Answer
“I led the transition from a monolith SPA to a micro-frontend architecture for a 50+ developer team. After evaluating iframes (performance and UX) and single-spa, we selected Webpack Module Federation because of its native lazy loading, dependency sharing, and backward compatibility. We defined a hypervisor (shell) that manages route table and renders remote components. We enforced a design system with shared component library and design tokens, ensuring visual consistency. To isolate CSS, we used CSS Modules and a naming convention. For state, we kept local state per micro-app, with a thin shared context (user info, permissions) via a custom event bus. We set up independent CI/CD pipelines for each micro-app, and integrated E2E tests across apps using Playwright. Performance optimization: shared vendor chunk, code splitting per route. We also had to manage versioning: shell with semver, micro-apps follow contract; breaking changes handled via versioned remote entries. This allowed teams to ship on their own cadence, reducing release conflicts and accelerating feature delivery. The main challenge was runtime errors from version mismatches; we built a dashboard to detect missing remotes and fallback gracefully. This architecture also improved onboarding—new teams can start a new micro-app quickly.”

### Topic: Testing Culture and Flaky Tests

#### ❌ Mid-level Answer
“We have unit tests and integration tests. Sometimes tests are flaky, we just retry.”

#### ✅ Senior Answer
“We categorize flaky tests and fix root causes: async timing, shared state, test ordering. We use quarantine to prevent blocking CI but track them. We enforce deterministic tests with proper teardown. We aim for a reliable test suite that developers trust.”

#### 🚀 Staff-level Answer
“I drove a quality initiative reducing test flakiness from 5% to <0.1%. We instrumented test results to identify flaky tests across runs, using analytics. Root causes included asynchronous assertions (missing `waitFor`), test databases with dirty state (solution: rollback transactions, test containers, unique prefixes), and third-party dependencies (solution: MSW to mock network). We introduced a flaky test quarantine: flaky tests are automatically quarantined after X failures, notifying the owning team to fix. We set a rule: zero tolerance for flaky tests in critical path. To maintain culture, we held brown-bags on testing best practices and recognized engineers who fixed the most flaky tests. This significantly shortened CI feedback loop and increased deployment confidence.”

## 5. High ROI Questions
- How would you design a reusable component library for a large organization?
- Explain the trade-offs between different state management patterns in React/Angular.
- What are Core Web Vitals and how do you measure/improve them?
- Compare SSR and CSR: when to use each?
- Micro-frontends: benefits, challenges, and implementation options.
- How do you ensure accessibility in a modern SPA?
- Describe your testing strategy from unit to production.
- How do you write testable code? (Frontend and backend)
- What is test-driven development, and have you practiced it effectively?
- How do you handle test data in integration tests?
- How would you set up a CI pipeline with automated quality gates?
- What is mutation testing and why is it useful?

**Tricky Follow-ups:**
- “Your E2E tests take 30 minutes; how do you reduce that while maintaining coverage?”
- “A critical test fails intermittently in CI but never locally; how do you debug?”
- “How do you test a real-time collaborative editing feature (like Google Docs)?”

## 6. Cheat Sheet (Revision)
- **Frontend perf**: lazy load, code split, optimize images, minimize main thread work, use web vitals.
- **Security**: CSP, httpOnly cookies for auth, validate inputs, escape outputs, XSS protection.
- **State**: Lift state up; use global stores sparingly; server state cache with React Query.
- **Testing pyramid**: more unit, fewer e2e; contract tests between services.
- **Flaky tests**: quarantine, root cause, fix; no retry without analysis.
- **Quality gates**: lint, test coverage on new code, no critical vulnerabilities, perf budget, visual regressions.

---

### 🎤 Communication Training
- **Frontend focus**: “I always consider the user experience and performance first. For example, we optimized LCP by 40% by implementing SSR for the critical content and lazy-loading below-the-fold components.”
- **Testing mindset**: “Testing is not just a QA activity; it’s a developer responsibility. I advocate for writing tests that give confidence to refactor, not just coverage numbers.”

### 📈 Personalised Self-Assessment (12 YOE)
- **Strong areas**: likely you’re strong in backend, but frontend may be weaker (unless you’re full-stack). Testing may be solid for backend.
- **Hidden gaps**: Modern frontend architecture (micro-frontends, module federation), deep web vitals optimization, advanced React patterns, accessibility, testing for frontend async behavior, visual regression tools.
- **Overconfidence risks**: Assuming you can design a frontend without knowing performance implications; neglecting accessibility; not having strategies for flaky tests; treating testing as an afterthought.

Next combined topic when you say "next".
