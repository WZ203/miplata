# Mi Plata — versión para GitHub Pages

Aplicación personal y privada para registrar gastos, revisar categorías y visualizar la evolución mensual.

## Publicarla en GitHub Pages

1. Crea un repositorio nuevo en GitHub.
2. Descomprime este paquete y sube **todo su contenido** a la rama `main`.
3. En el repositorio, abre **Settings → Pages**.
4. En **Source**, selecciona **GitHub Actions**.
5. Abre la pestaña **Actions** y espera a que termine “Publicar Mi Plata”.
6. GitHub mostrará la dirección pública de la aplicación.

Cada vez que modifiques la rama `main`, la web se publicará nuevamente de manera automática.

## Privacidad y seguridad

- Los gastos no se guardan en GitHub.
- Se cifran con tu clave y permanecen en el navegador del dispositivo.
- GitHub Pages utiliza HTTPS, necesario para el cifrado.
- Si cambias de teléfono o borras los datos del navegador, deberás restaurar un respaldo descargado desde Configuración.
- Aunque el repositorio sea público, tus gastos no estarán dentro del código.

## Probarla en el computador

Necesitas Node.js 22 o superior.

```bash
npm install
npm run dev
```

## Compilar manualmente

```bash
npm run build
```

Los archivos estáticos se generarán dentro de `dist`.
