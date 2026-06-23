# 🧪 Estrategia de Testing

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> Ya sabes *escribir* tests con [[Desarrollo Profesional/Python/12 - Testing con Pytest|pytest]]. Esta sección es sobre algo distinto y más difícil: la *estrategia*. Qué testear y qué no, en qué nivel poner cada test, cómo equilibrar confianza contra velocidad y mantenimiento. Un proyecto con 5.000 tests lentos y frágiles puede dar menos confianza que uno con 500 bien pensados. El objetivo es construir una suite que detecte regresiones de verdad, corra rápido y no se rompa con cada refactor — para que puedas desplegar con confianza ([[Desarrollo Profesional/CI-CD e IaC/CI-CD e IaC|CI/CD]]).

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/Estrategia de Testing/Páginas/01 - Pirámide y TDD|01 — Pirámide y TDD]] — la pirámide de tests, TDD, qué es un buen test, mocks y dobles
2. [[Desarrollo Profesional/Estrategia de Testing/Páginas/02 - Tipos de Test y Estrategia|02 — Tipos de Test y Estrategia]] — integración, e2e, contract, mutation, cobertura y su lectura correcta
3. [[Desarrollo Profesional/Estrategia de Testing/Páginas/03 - TDD en Profundidad y Kata|03 — TDD en Profundidad y Kata]] — kata completa paso a paso, outside-in vs inside-out, TDD en legacy

---

## ¿Qué resuelve esta sección en una frase?

Te da los criterios para construir una suite de tests que dé confianza real al mejor coste posible de velocidad y mantenimiento — distinguiendo los tests que protegen comportamiento de los que solo frenan los refactors.

---
`#testing` `#calidad` `#tdd` `#indice`
