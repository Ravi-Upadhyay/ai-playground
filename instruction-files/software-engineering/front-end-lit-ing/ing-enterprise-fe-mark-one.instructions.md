---
name: ing-enterprise-fe
description: "Enterprise-grade front-end JavaScript guidelines for ING consumer loan projects. Lit web components, Lion-derived components, proprietary Design System integration, accessibility (WCAG 2.1 AA), security (XSS/CSP/input validation), performance, testability, and Azure DevOps CI/CD. Use when building/reviewing web components, writing unit tests, integrating design tokens, or implementing secure forms."
---

# Enterprise Front-End JavaScript Guidelines

**Scope**: Modern JS, Web Components (Lit), Lion-derived + custom components, proprietary Design System, Node tooling, Azure DevOps CI/CD  
**Audience**: ING consumer loan front-end team  
**Updated**: April 2026

---

## Decision Framework

**When uncertain, choose in this order**:
1. Standards-based (W3C, WHATWG, WAI-ARIA)
2. Security/accessibility first
3. Testability
4. Developer ergonomics

---

## Architecture & Component Patterns

### Component Classification

| Type | Source | Pattern | Use When |
|------|--------|---------|----------|
| **Design System** | Proprietary CSS web components | Import + extend via Lit | Visual primitives (button, input, card) |
| **Lion-Derived** | Forked from Lion Input/Form | Custom controller + Lit mixins | Form inputs, validation logic needed |
| **Custom Components** | Scratch build | Pure Lit + composition | Domain logic, unique interactions |

### Naming & File Structure

**Classes** (PascalCase, match file name):
```javascript
// ConsumerLoanForm.js
export class ConsumerLoanForm extends LitElement { }
```

**Elements** (kebab-case, `ing-` prefix):
```javascript
customElements.define('ing-consumer-loan-form', ConsumerLoanForm);
```

**Derived from Lion** (indicate lineage):
```javascript
// LoanAmountInput.js — derived from LionInput
export class LoanAmountInput extends LionInput { }
customElements.define('ing-loan-amount-input', LoanAmountInput);
```

**File co-location**:
```
src/ConsumerLoanForm/
├── ConsumerLoanForm.js         (component)
├── ConsumerLoanForm.test.js    (unit tests — ALWAYS)
├── ConsumerLoanForm.styles.js  (extracted styles if >100 lines)
└── ConsumerLoanForm.stories.js (Storybook documentation)
```

---

## Security: XSS, CSP, Input Validation

### XSS Prevention

**Rule**: Never use `innerHTML` or `dangerouslySetInnerHTML`. Always use Lit's `html` template.

```javascript
// ✓ CORRECT: Lit templates auto-escape
render() {
  return html`<p>${this.userInput}</p>`;
}

// ✗ WRONG: Creates XSS vulnerability
render() {
  return html`<p>${unsafeHTML(this.userInput)}</p>`;
}

// ✗ WRONG: Direct DOM manipulation
render() {
  const el = document.createElement('div');
  el.innerHTML = this.userInput; // XSS!
  return el;
}
```

**Sanitization for rare cases** (rich text editors):
- Use `DOMPurify` with strict config
- Never trust user/external data
- Whitelist tags explicitly

```javascript
import DOMPurify from 'dompurify';

const clean = DOMPurify.sanitize(userHtml, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p'],
  ALLOWED_ATTR: [],
});
```

### CSP Compliance

**Inline styles/event handlers break CSP**:

✓ CORRECT:
```javascript
static styles = css`
  :host { color: var(--ing-text-primary); }
`;

render() {
  return html`<button @click="${this._handleClick}">Action</button>`;
}
```

✗ WRONG:
```javascript
render() {
  return html`<button style="color: red" onclick="alert('hi')">Action</button>`;
}
```

### Input Validation

**Validate on entry** (client + server):

```javascript
import { LitElement, html } from 'lit';

export class LoanAmountInput extends LitElement {
  #validator = {
    min: 1000,
    max: 1000000,
    pattern: /^[0-9]+$/, // digits only
  };

  _handleInput(e) {
    const value = e.target.value;
    
    // 1. Type check
    if (!/^\d+$/.test(value)) {
      this._setError('Numbers only');
      return;
    }
    
    // 2. Range check
    const num = parseInt(value, 10);
    if (num < this.#validator.min || num > this.#validator.max) {
      this._setError(`Amount between €${this.#validator.min} and €${this.#validator.max}`);
      return;
    }
    
    // 3. Update state
    this.loanAmount = num;
    this._clearError();
    
    // 4. Fire for parent to validate (don't trust client-side alone)
    this.dispatchEvent(new CustomEvent('amount-changed', { detail: num }));
  }

  _setError(msg) {
    this.validationError = msg;
    this.requestUpdate();
  }

  _clearError() {
    this.validationError = null;
  }
}
```

**Never silently ignore invalid input** — always provide feedback + log for audit.

---

## Accessibility (WCAG 2.1 Level AA)

### Semantic HTML First

```javascript
// ✓ CORRECT: Use native elements, extend with ARIA where needed
render() {
  return html`
    <form role="form" aria-label="Consumer Loan Application">
      <label for="amount">Loan Amount</label>
      <input
        id="amount"
        type="number"
        min="1000"
        max="1000000"
        aria-describedby="amount-hint"
        required
      />
      <small id="amount-hint">Between €1,000 and €1,000,000</small>
    </form>
  `;
}

// ✗ WRONG: Divs + custom event handling
render() {
  return html`
    <div role="form">
      <div>Loan Amount</div>
      <div @click="${this._handleClick}">Input field</div>
    </div>
  `;
}
```

### Focus Management & Keyboard

```javascript
export class ConsumerLoanForm extends LitElement {
  // 1. Always set tabindex for interactive elements
  render() {
    return html`
      <button tabindex="0" @click="${this._submit}">Submit</button>
      <input tabindex="0" type="text" />
    `;
  }

  // 2. Handle Escape key
  _handleKeydown(e) {
    if (e.key === 'Escape') {
      this._closeDialog();
    }
  }

  // 3. Restore focus after modal closes
  _closeDialog() {
    this._lastFocusedElement?.focus();
  }

  connectedCallback() {
    super.connectedCallback();
    this._lastFocusedElement = document.activeElement;
  }
}
```

### ARIA Labels & Live Regions

```javascript
render() {
  return html`
    <!-- Error messages in live region -->
    <div aria-live="polite" aria-atomic="true" role="alert">
      ${this.errorMessage || ''}
    </div>

    <!-- Form validation status -->
    <input
      aria-label="Loan amount in euros"
      aria-invalid="${this.hasError}"
      aria-describedby="${this.hasError ? 'error-msg' : ''}"
    />
  `;
}
```

**Automated Testing**: Use `@storybook/addon-a11y` + axe-core in CI/CD.

---

## Testability & Unit Testing (Web Test Runner)

### Test-First Thinking

**Write tests BEFORE implementation** for:
- Input validation logic
- State transitions
- Event emissions
- Integration with Design System components

### Component Test Template

```javascript
// ConsumerLoanForm.test.js
import { html, fixture, expect, waitUntil } from '@open-wc/testing';
import sinon from 'sinon';
import '../src/ConsumerLoanForm.js';

describe('ConsumerLoanForm', () => {
  let element;

  beforeEach(async () => {
    element = await fixture(
      html`<ing-consumer-loan-form></ing-consumer-loan-form>`
    );
  });

  describe('Initialization', () => {
    it('renders with default state', () => {
      expect(element.loanAmount).to.equal(0);
      expect(element.validationError).to.be.null;
    });

    it('initializes from Design System tokens', () => {
      const style = getComputedStyle(element);
      expect(style.getPropertyValue('--ing-text-primary')).to.not.be.empty;
    });
  });

  describe('Input Validation', () => {
    it('rejects non-numeric input', () => {
      element._handleInput({ target: { value: 'abc' } });
      expect(element.validationError).to.include('Numbers only');
    });

    it('enforces min/max bounds', () => {
      element._handleInput({ target: { value: '5000000' } });
      expect(element.validationError).to.include('€1,000');
    });

    it('clears error on valid input', () => {
      element._handleInput({ target: { value: '50000' } });
      expect(element.validationError).to.be.null;
    });
  });

  describe('Event Emission', () => {
    it('fires amount-changed on valid input', async () => {
      const spy = sinon.spy();
      element.addEventListener('amount-changed', spy);
      
      element._handleInput({ target: { value: '50000' } });
      await element.updateComplete;
      
      expect(spy.calledOnce).to.be.true;
      expect(spy.firstCall.detail).to.equal(50000);
    });

    it('does NOT fire on invalid input', () => {
      const spy = sinon.spy();
      element.addEventListener('amount-changed', spy);
      
      element._handleInput({ target: { value: 'invalid' } });
      
      expect(spy.called).to.be.false;
    });
  });

  describe('Design System Integration', () => {
    it('uses proprietary Design System button component', async () => {
      const btn = element.shadowRoot.querySelector('ing-ds-button');
      expect(btn).to.exist;
    });

    it('respects Design System theme tokens', () => {
      const style = getComputedStyle(element.shadowRoot.querySelector('button'));
      // Verify CSS custom property is applied
      expect(style.backgroundColor).to.include('rgb');
    });
  });

  describe('Accessibility', () => {
    it('has accessible form labels', () => {
      const label = element.shadowRoot.querySelector('label[for="amount"]');
      expect(label?.textContent).to.include('Loan Amount');
    });

    it('announces validation errors to screen readers', async () => {
      element._handleInput({ target: { value: 'invalid' } });
      await element.updateComplete;
      
      const alert = element.shadowRoot.querySelector('[role="alert"]');
      expect(alert?.getAttribute('aria-live')).to.equal('polite');
      expect(alert?.textContent).to.include('Numbers only');
    });

    it('supports keyboard navigation', () => {
      const input = element.shadowRoot.querySelector('input');
      expect(input?.getAttribute('tabindex')).to.not.be.null;
    });
  });
});
```

### Running Tests

```bash
npm test                      # Once
npm run test:watch           # Watch mode
npm run test:coverage        # Coverage report

# CI/CD (Azure DevOps)
npm run test:ci              # Generates XML for pipeline
```

### Coverage Targets

- **Statements**: ≥ 80%
- **Branches**: ≥ 75%
- **Lines**: ≥ 80%
- **Functions**: ≥ 80%

Fail CI/CD below thresholds.

---

## Performance

### Bundle Size

```bash
npm run build              # Generates rollup output
npm run analyze:build      # Check bundle size
```

**Target**: Single component < 15KB (gzipped)

**Anti-patterns**:
- ✗ Importing entire libraries in templates
- ✗ Large external dependencies without lazy loading
- ✗ Unoptimized CSS in shadow DOM

### Render Performance

```javascript
export class ConsumerLoanForm extends LitElement {
  // 1. Minimize reactivity scope — use private fields for non-reactive data
  #internalCache = {};

  static properties = {
    loanAmount: { type: Number }, // Only what needs to trigger re-renders
  };

  // 2. Use @query for one-time DOM access
  @query('#amount-input')
  _amountInput;

  // 3. Memoize expensive calculations
  get interestAmount() {
    if (this._cachedAmount === this.loanAmount) {
      return this._cachedInterest;
    }
    this._cachedAmount = this.loanAmount;
    this._cachedInterest = this.loanAmount * 0.035; // 3.5%
    return this._cachedInterest;
  }

  // 4. Defer non-critical DOM updates
  render() {
    return html`
      <form>
        ${this._renderCritical()} <!-- Fast path -->
        ${this._renderSupportingInfo()} <!-- Can wait -->
      </form>
    `;
  }

  _renderCritical() {
    return html`<input value="${this.loanAmount}" />`;
  }

  _renderSupportingInfo() {
    return html`<p>Interest: €${this.interestAmount}</p>`;
  }
}
```

---

## Design System Integration

### CSS Token Usage

```javascript
import { LitElement, html, css } from 'lit';

export class ConsumerLoanForm extends LitElement {
  static styles = css`
    :host {
      /* Always use proprietary Design System tokens */
      --color-primary: var(--ing-ds-primary, #003399);
      --color-text: var(--ing-ds-text, #212121);
      --font-family: var(--ing-ds-font-family, sans-serif);
    }

    .form {
      color: var(--color-text);
      font-family: var(--font-family);
    }

    button {
      background: var(--color-primary);
      padding: var(--ing-ds-spacing-md, 12px);
      border-radius: var(--ing-ds-radius, 4px);
    }
  `;

  render() {
    return html`
      <form class="form">
        <button>Submit</button>
      </form>
    `;
  }
}
```

### Importing Design System Components

```javascript
// Components already follow Design System
import 'proprietary-ds/components/button.js';
import 'proprietary-ds/components/input.js';
import 'proprietary-ds/components/card.js';

export class ConsumerLoanForm extends LitElement {
  render() {
    return html`
      <ing-ds-card>
        <ing-ds-input label="Amount"></ing-ds-input>
        <ing-ds-button>Apply</ing-ds-button>
      </ing-ds-card>
    `;
  }
}
```

---

## Code Quality & Anti-Patterns

### ✗ Anti-Patterns (Call Out Strictly)

| Anti-Pattern | Why Wrong | Fix |
|---|---|---|
| Using `querySelector` in lifecycle hooks without null-check | Race conditions, timing issues | Use `@query` decorator or guard in `updated()` |
| Mixing validation logic in render | Untestable, couples presentation to business logic | Extract to pure functions, test separately |
| Storing DOM references in properties | Memory leaks on re-render | Use `@query` or create in render |
| Direct `dataset` manipulation | Loses type safety, security risk | Use properties + Lit reactivity |
| Not handling async errors in event handlers | Silent failures in production | Always add `try/catch` + error logging |
| Using ES3 inheritance with Lit | Mixins don't work, TypeScript breaks | Use composition or Lit mixins only |

### ✓ Code Patterns

**Composition over Inheritance**:
```javascript
// ✓ CORRECT: Composition
const validationMixin = (Base) => class extends Base {
  validate() { }
};

export class ConsumerLoanForm extends validationMixin(LitElement) { }

// ✗ WRONG: Deep inheritance chains
export class ConsumerLoanForm extends LionForm { } // Leaks Lion internals
```

**Pure Functions for Logic**:
```javascript
// ✓ CORRECT: Testable, reusable
export const calculateMonthlyPayment = (principal, annualRate, months) => {
  const r = annualRate / 100 / 12;
  return (principal * r * Math.pow(1 + r, months)) / (Math.pow(1 + r, months) - 1);
};

// In component:
const monthly = calculateMonthlyPayment(this.loanAmount, this.interestRate, 180);

// ✗ WRONG: Tied to component
_calculateMonthlyPayment() {
  this.monthly = this.loanAmount * ... // Untestable
}
```

**Error Handling**:
```javascript
async _submitForm() {
  try {
    const response = await this._validateAndSubmit();
    this.dispatchEvent(new CustomEvent('submit-success', { detail: response }));
  } catch (error) {
    // Log for auditing (security)
    console.error('[FE-LOAN-FORM]', error.message);
    
    // Show user-friendly message
    this.validationError =  'Unable to process request. Please try again.';
  }
}
```

---

## Azure DevOps CI/CD Integration

### Pre-commit Hooks

```bash
# .husky/pre-commit
npm run lint:fix
npm test -- --coverage
```

**Fail if**:
- ESLint violations
- Test coverage < 80%
- Type errors (if using TypeScript)

### Pipeline Configuration

```yaml
# azure-pipelines.yml
trigger:
  - main
  - feature/*

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '18.x'

  - script: npm ci
    displayName: 'Install dependencies'

  - script: npm run lint
    displayName: 'Lint code'

  - script: npm run test:ci
    displayName: 'Run unit tests'

  - task: PublishTestResults@2
    inputs:
      testResultsFiles: 'coverage/test-results.xml'
    condition: always()

  - task: PublishCodeCoverageResults@1
    inputs:
      codeCoverageTool: 'Cobertura'
      summaryFileLocation: 'coverage/cobertura-coverage.xml'

  - script: npm run build
    displayName: 'Build for production'

  - script: npm run analyze:build
    displayName: 'Analyze bundle'
```

---

## VS Code + Copilot Best Practices

### Copilot Prompting

✓ GOOD:
```
"Create a login form component with email validation, password strength indicator, 
and accessibility. Include unit tests for validation logic and a11y checks."
```

✗ VAGUE:
```
"Write a form component"
```

### Debugging in VS Code

```javascript
// Set breakpoints, then test
element.loanAmount = 50000;
element.requestUpdate();
await element.updateComplete;
// Step through _handleInput() with debugger
```

### Extensions

- **Lit** (runem.lit-plugin): Auto-completion in templates
- **ESLint** (dbaeumer.vscode-eslint)
- **Web Test Runner** (coverage reports)
- **axe DevTools** (accessibility audit)

---

## Quick Reference

### Commands

| Task | Command |
|------|---------|
| Start dev | `npm start` |
| Test once | `npm test` |
| Test watch | `npm run test:watch` |
| Lint + fix | `npm run lint:fix` |
| Build | `npm run build` |
| Check bundle | `npm run analyze:build` |
| Storybook | `npm run storybook` |
| Pre-commit | `npm run prepare` |

### Checklist Before PR

- [ ] Unit tests written (80%+ coverage)
- [ ] Accessibility tested (@a11y addon)
- [ ] No XSS vulnerabilities (Lit `html` only)
- [ ] Input validation + error messages
- [ ] Design System tokens used (not hardcoded colors)
- [ ] No console errors/warnings
- [ ] Bundle size < 15KB gzipped
- [ ] ESLint clean (`npm run lint`)
- [ ] Keyboard navigation works
- [ ] No sensitive data logged

### File Checklist

```
src/ComponentName/
✓ ComponentName.js              (component class)
✓ ComponentName.test.js         (unit tests, 80%+ coverage)
✓ ComponentName.stories.js      (Storybook, a11y tests)
✓ ComponentName.styles.js       (if styles > 100 lines)
? ComponentName.utils.js        (if pure helpers extracted)
```

---

## Escalation & Redirect

**This guide covers**: Lit + Lion-derived components, Design System integration, test-driven development, security/accessibility patterns.

**Redirect to separate chat if**: AWS/Azure DevOps infrastructure, monorepo strategy, package publishing, advanced TypeScript generics, designer collaboration.

---

**Last Updated**: April 2026  
**Project**: ING Consumer Loan (P01647)  
**Standard**: WCAG 2.1 AA, OWASP Top 10, Lit best practices
