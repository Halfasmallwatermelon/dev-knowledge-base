# 端到端测试指南

## Cypress

```javascript
describe('Login', () => {
  it('should login successfully', () => {
    cy.visit('/login')
    cy.get('input[name="email"]').type('user@example.com')
    cy.get('input[name="password"]').type('password123')
    cy.get('button[type="submit"]').click()
    cy.url().should('include', '/dashboard')
  })
  
  it('should show error for invalid credentials', () => {
    cy.visit('/login')
    cy.get('input[name="email"]').type('wrong@example.com')
    cy.get('input[name="password"]').type('wrongpassword')
    cy.get('button[type="submit"]').click()
    cy.get('.error-message').should('be.visible')
  })
})

// API 测试
describe('API', () => {
  it('should get users', () => {
    cy.request('GET', '/api/users').then((response) => {
      expect(response.status).to.eq(200)
      expect(response.body).to.be.an('array')
    })
  })
})
```

## Playwright

```typescript
import { test, expect } from '@playwright/test'

test('homepage has title', async ({ page }) => {
  await page.goto('http://localhost:3000')
  await expect(page).toHaveTitle(/My App/)
})

test('login flow', async ({ page }) => {
  await page.goto('/login')
  await page.fill('input[name="email"]', 'user@example.com')
  await page.fill('input[name="password"]', 'password123')
  await page.click('button[type="submit"]')
  await expect(page).toHaveURL('/dashboard')
})
```