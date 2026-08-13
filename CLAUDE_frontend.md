# Frontend, React + Vite

## Layout
    src/main.jsx
    src/App.jsx
    src/api/         fetch wrappers, one file per resource. All network code lives here.
    src/components/  reusable components
    src/pages/       route level components
    src/styles/      global.css and shared CSS Module partials

## Rules
- CSS Modules only. No Tailwind, no styled-components, no inline style objects
  except for values that are genuinely dynamic at runtime.
- A component lives next to its styles:
  components/QuizCard/QuizCard.jsx and components/QuizCard/QuizCard.module.css
- Plain JavaScript, function components, hooks. No class components.
- No global state library. useState and useContext until that actually hurts.
- Components never call fetch directly. They import from src/api/.
- Base URL comes from import.meta.env.VITE_API_URL.
- Keep components under roughly 150 lines. Split when they grow past that.
- Colors, spacing, and radii are CSS custom properties defined in src/styles/global.css.