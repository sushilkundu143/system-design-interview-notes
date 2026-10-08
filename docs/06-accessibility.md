# Accessibility: Interview Guide

## 1. What accessibility means

Accessibility means people can perceive, understand, navigate, and operate an
application regardless of disabilities or input methods. It includes people
using screen readers, keyboards, magnification, voice input, and switch devices,
as well as people affected by temporary injuries or situational limitations.

It is a product-quality requirement, not a final automated scan.

For a banking application, a person must be able to select an account, understand
a balance, correct transfer errors, and confirm the outcome without relying on
vision, a mouse, or color alone.

## 2. WCAG and POUR

WCAG is the Web Content Accessibility Guidelines. WCAG 2.2 is a useful current
reference; an organization's contractual target may still specify WCAG 2.1 AA.
Check the exact legal and product requirements rather than claiming compliance
based on one tool.

| Principle | Meaning | Example |
| --- | --- | --- |
| Perceivable | Information can be perceived through suitable alternatives | Text alternatives and sufficient contrast |
| Operable | Controls and navigation can be operated | Keyboard support and no focus traps |
| Understandable | Content and behavior are predictable | Clear labels and actionable errors |
| Robust | Content works with user agents and assistive technology | Correct names, roles, and states |

Conformance levels are A, AA, and AAA. AA conformance includes applicable A and
AA criteria; it is not simply an average accessibility score.

## 3. Start with semantic HTML

Use an element that already has the required behavior:

```jsx
<button type="button" onClick={openDetails}>
  View account details
</button>
```

A clickable `div` does not automatically have button semantics, keyboard
activation, or focusability. Adding `role="button"` alone does not supply them.

Use:

- Buttons for actions; links for navigation.
- Headings that describe the document structure.
- `main`, `nav`, and other appropriate landmarks.
- Labels for form controls.
- Tables for genuinely tabular data, with header relationships.
- Lists for related collections.

Prefer native behavior over rebuilding it with ARIA.

## 4. Accessible names, roles, and states

Assistive technology needs to know what a control is, what it does, and its
current state.

```jsx
<button
  type="button"
  aria-expanded={expanded}
  aria-controls="account-details"
  onClick={() => setExpanded((value) => !value)}
>
  Account details
</button>
<section id="account-details" hidden={!expanded}>
  {/* Account information */}
</section>
```

Visible text often provides the best accessible name. Use `aria-label` for an
otherwise unnamed control, such as an icon-only close button. Avoid overriding
clear visible text unnecessarily.

Use `aria-describedby` for additional instructions or errors. Keep names
compatible with visible labels so voice-control users can reference them.

ARIA exposes semantics; it does not implement keyboard interaction.

## 5. Keyboard and focus management

### Essential behaviors

- All interactive features are reachable and operable.
- Tab order follows a logical sequence.
- Focus is visible and not hidden behind sticky elements or overlays.
- There is no unintended keyboard trap.
- Avoid positive `tabIndex` values that create a separate artificial order.
- Offer a skip link when users otherwise repeat extensive navigation.

Composite widgets such as tabs and menus have specific arrow-key and focus
patterns. Follow an established accessible component rather than treating every
item as a generic button.

### Dialog lifecycle

1. Opening a modal moves focus inside to an appropriate element.
2. Tab navigation remains within the modal while it is active.
3. Background content cannot be interacted with as though it were active.
4. The dialog has an accessible name.
5. Escape normally closes it, subject to justified workflow constraints.
6. Closing restores focus to the trigger or another logical destination.

Use a tested dialog implementation or correctly managed native dialog. Portals
alone do not implement accessibility.

### SPA route changes

A client-side navigation does not inherently reproduce a full document load.
Update the document title, preserve meaningful navigation state, and manage
focus or route announcements according to the application's UX.

Do not forcibly move focus on every background update.

## 6. Accessible forms and errors

```jsx
<label htmlFor="amount">Transfer amount</label>
<input
  id="amount"
  name="amount"
  inputMode="decimal"
  value={amount}
  onChange={(event) => setAmount(event.target.value)}
  aria-invalid={Boolean(error)}
  aria-describedby={error ? "amount-help amount-error" : "amount-help"}
/>
<p id="amount-help">Enter the amount in INR.</p>
{error && <p id="amount-error">{error}</p>}
```

This is a teaching fragment, not a complete transfer-validation implementation.

Good form behavior:

- Do not use placeholders as the only labels.
- Group related controls with appropriate legends.
- Explain formats and requirements before submission.
- Identify errors in text, not just a red border.
- Preserve valid input after failed submission.
- For multiple errors, provide an error summary linked to the relevant fields.
- Avoid announcing validation on every keystroke.
- Provide review/correction safeguards for financial submissions.

Server-side validation remains necessary for correctness and security.

## 7. Visual, motion, and content requirements

Common WCAG AA considerations:

- Normal text generally needs at least **4.5:1** contrast.
- Large text generally needs at least **3:1** contrast under WCAG's definition.
- Relevant non-text UI boundaries and states generally need **3:1** contrast,
  subject to the criterion's scope and exceptions.
- Color must not be the only way to convey meaning.
- Support text resizing and reflow; test narrow layouts without losing controls.
- WCAG 2.2 adds a **24 by 24 CSS pixel** minimum target-size criterion with
  exceptions and spacing alternatives; larger targets often improve usability.
- Respect reduced-motion preferences and provide applicable media controls.
- Supply captions/transcripts or alternatives according to the media type.

For images, use meaningful alternative text when they convey information.
Decorative images generally use an empty `alt`. Do not repeat adjacent text
without adding value.

## 8. Async status and dynamic interfaces

Use a suitable status announcement for meaningful completion:

```jsx
<p role="status">{saveMessage}</p>
```

A status region provides polite announcements without moving focus. Ensure the
region and updates behave correctly with the assistive technology you support.
Reserve assertive alerts for genuinely urgent information.

Loading placeholders should not create noisy announcements or confusing
controls. Disable or guard duplicate actions while explaining pending state.

Virtualized lists can make content unavailable to assistive technology or
keyboard navigation if implemented carelessly. Test reading order, focus
retention, and access to items beyond the initial viewport.

## 9. Testing strategy

Use multiple layers:

| Layer | What it catches |
| --- | --- |
| Component linting | Common markup and naming mistakes |
| Automated scans such as axe | Detectable structural/contrast issues |
| Keyboard testing | Focus order, activation, traps, and visibility |
| Screen-reader testing | Names, announcements, relationships, and reading order |
| Zoom/reflow testing | Clipped content and unusable layouts |
| User testing | Real task completion and usability |

React Testing Library encourages role/name queries:

```jsx
await user.click(
  screen.getByRole("button", { name: "Save preferences" })
);
expect(await screen.findByRole("status")).toHaveTextContent(
  "Preferences saved"
);
```

Assume the test renders the feature with its required dependencies.

Automated checks cannot prove accessibility. A correctly named button can still
be unreachable inside a broken modal.

## 10. Enterprise process

Include accessibility in acceptance criteria and the design system. Test shared
controls once thoroughly and also verify their actual page composition.

Example transfer acceptance criteria:

1. All steps are keyboard-operable.
2. Source/destination controls have unambiguous labels.
3. Validation errors are associated with their fields.
4. Review and confirmation are understandable with a screen reader.
5. Focus returns logically after dialogs close.
6. The complete flow works at supported zoom and viewport sizes.

Track issues with user impact and ownership. Do not allow a known baseline to
become a permanent excuse for new regressions.

## 11. Interview questions

**Is ARIA better than HTML?**

No. Use native semantics first; use ARIA where additional semantics are needed.
Incorrect ARIA can make an interface worse.

**How do you make a React application accessible?**

Design semantic components, correct keyboard/focus behavior, understandable
forms, and meaningful async feedback; verify automated and manual task flows.

**Can Lighthouse prove WCAG conformance?**

No. It provides useful automated evidence but does not evaluate every criterion
or complete user experience.

**How do accessibility and performance interact?**

Fast responses help usability, but optimization must preserve focus, reading
order, and access to content. Virtualization requires particular care.

## 12. Interview summary

> I build accessibility into component design and acceptance criteria using
> semantic HTML, accessible names, keyboard/focus management, and clear errors.
> I combine automated checks with keyboard, screen-reader, and complete-flow
> testing rather than claiming compliance from a tool score.

## References

- [WCAG 2.2](https://www.w3.org/TR/WCAG22/)
- [WAI ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- [React accessibility-related DOM props](https://react.dev/reference/react-dom/components/common)
- [axe-core](https://github.com/dequelabs/axe-core)
- [Testing Library queries](https://testing-library.com/docs/queries/about/)
