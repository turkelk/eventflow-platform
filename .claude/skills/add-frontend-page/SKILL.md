# Add Frontend Page

## Steps
1. **Component** — Create `src/pages/<PageName>.tsx`.
2. **Route** — Register in the router (React Router).
3. **API hook** — Create `src/hooks/use<Resource>.ts` for data fetching with React Query.
4. **Loading state** — Add skeleton or spinner while data loads.
5. **Error state** — Handle API errors with a user-friendly message.

## Styling
- Use Tailwind utility classes. Avoid custom CSS unless absolutely necessary.
- Use responsive prefixes: `sm:`, `md:`, `lg:` for breakpoints.
- Add `dark:` variants for all colors. Test in both light and dark mode.
- Use shadcn/ui components as the base — customize with Tailwind.

## Responsive
- Mobile-first: design for small screens, then add breakpoints.
- Test at 320px, 768px, and 1024px minimum.
- Ensure touch targets are at least 44x44px.

## Accessibility
- Use semantic HTML (`main`, `nav`, `section`, `article`, `h1-h6`).
- Add `aria-label` to icon-only buttons.
- Ensure keyboard navigation works (Tab, Enter, Escape).
- Test with a screen reader.