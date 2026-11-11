# Auro Consultorio - demo

Prototipo visual de una agenda para consultorio, desarrollado con SvelteKit, Svelte 5 y TypeScript.

**Solo demostracion.** Todos los pacientes y turnos son ficticios. Los cambios se guardan unicamente en memoria, se pierden al recargar y no existe backend ni historial clinico real.

## Desarrollo local

```sh
npm ci
npm run dev
```

## Verificaciones

```sh
npm run check
npm run build
```

## Publicacion

El workflow `.github/workflows/deploy.yml` compila la demo estatica y la publica automaticamente en GitHub Pages cada vez que se envia un commit a `main`.

GitHub Pages es un hosting publico para esta demostracion, **no** un servidor para pacientes, historiales ni datos medicos reales.

Los materiales de estudio y gestion interna (.mssr, docs, .study, notas privadas) no forman parte de este repositorio.

