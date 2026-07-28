# lame-content-data

Datos públicos del **contenido editorial** de la app *La Misa Explicada* (iOS
y Android): artículos, bloques, glosario, documentos e imágenes producidos en
el editor web (`misa-editorV2-swift`). El app consulta el
`content-manifest.json` de este repo y descarga el bundle e imágenes que hayan
cambiado, sin necesidad de una nueva versión en App Store / Play Store.

Ver `CONTENIDO_OTA_DISENO.md` (repo `SWIFT`) para el diseño completo. Este
repo es el análogo, para contenido editorial, de `lame-liturgia-data` (que
cubre el dataset **litúrgico**: calendario, lecturas, Liturgia de las Horas).
Son repos separados a propósito: cadencias de publicación independientes (el
"latest" de GitHub Releases es singular por repo — compartirlo obligaría a
re-publicar el dataset litúrgico completo en cada corrección editorial menor).

## Qué hay aquí (y qué NO)

**Sí:** `content_bundle.json` (secciones, definiciones, documentos, módulos,
`APP_INFO`) y las imágenes editoriales referenciadas por esos bloques
(`image`, `imgslider`, `flipcard`, etc.).

**No:** las imágenes del widget "Hoy" (`WIDGET_HOY_*.jpg`) ni las de
compartir (`SHARE_*`) — quedan fuera del alcance OTA v1 a propósito (el
widget las resuelve desde su propio bundle empacado, ver diseño). Tampoco el
texto litúrgico ni el calendario — eso vive en `lame-liturgia-data`.

Este repo es **público por necesidad, no por descuido**: el app lo descarga
sin credenciales. Un repo privado exigiría incrustar un token en el binario
(trivialmente extraíble, con rate limits compartidos entre todas las
instalaciones, y rotarlo requeriría un release de tienda — justo lo que este
mecanismo busca evitar). Es un artefacto de distribución: expone exactamente
los mismos bytes que cualquiera obtiene instalando la app gratis.

## Archivos

| Archivo | Qué es |
|---|---|
| `content-manifest.json` | Índice: `contentVersion`, `minSchema`, y el `file`/`bytes`/`sha256` del bundle y de cada imagen vigente. El app lo lee primero. |
| `content_bundle.json` | El contenido editorial completo (mismo artefacto que empaca el app, `Resources/content_bundle.json` en iOS / `assets/` en Android). |
| `*.jpg` / `*.png` / `*.webp` | Imágenes editoriales referenciadas por el bundle. |

`baseUrl` en el manifest apunta al Release donde viven los archivos; el app
arma cada URL como `baseUrl + file` y verifica el `sha256` tras descargar.

## Contrato del manifest

```json
{
  "contentVersion": 1,
  "minSchema": 1,
  "generatedAt": "2026-07-27",
  "baseUrl": "https://github.com/carlos-carmona-777/lame-content-data/releases/latest/download/",
  "bundle": { "file": "content_bundle.json", "bytes": 450670, "sha256": "…" },
  "images": [ { "file": "foo.jpg", "bytes": 12345, "sha256": "…" }, "…" ]
}
```

- `contentVersion`: entero monotónico. El app no aplica un manifest cuya
  versión sea `<=` la instalada.
- `minSchema`: gate de compatibilidad — si es mayor que el schema que soporta
  el binario instalado, el app ignora ese manifest (el contenido usa un tipo
  de bloque que ese binario no sabe renderizar). Solo sube a mano cuando el
  contenido use un tipo de bloque/campo estructuralmente nuevo.
- `images` es el **inventario completo** de imágenes vigentes, no solo las
  que cambiaron — el diff lo hace el app comparando `sha256` contra lo que
  ya tiene en su carpeta escribible local.

## Cómo publicar una versión nueva

1. En el repo `SWIFT` (o donde viva el pipeline del editor), regenerar el
   bundle con el flujo actual (`node convert.mjs …` → copiar a
   `Resources/content_bundle.json` de la app).
2. Correr el publicador:
   ```
   node migracion/publish-content.mjs
   ```
   Esto calcula `sha256` del bundle y de cada imagen en `Resources/Images/`,
   sube `contentVersion` en 1 respecto al manifest anterior de este repo, y
   copia manifest + bundle + imágenes nuevas/cambiadas aquí.
3. Revisar el diff (`git status` / `git diff`) y hacer commit en este repo.
4. `git push` — **esto publica a todos los usuarios con el toggle de
   contenido OTA activo** (y a todos, cuando el default pase a encendido).
   Pedir OK explícito antes de este push salvo que el usuario disponga lo
   contrario para este repo.
5. El CI (`.github/workflows/publish.yml`) crea el Release automáticamente al
   detectar el push a `main`; `"latest"` pasa a apuntar ahí.

## Regenerar / mantener

El bundle se genera en `migracion/convert.mjs` (repo `SWIFT`) a partir del
`content.js`/`documents.js` que exporta el editor
(`misa-editorV2-swift`). Los fixes de contenido (texto, imágenes) se hacen
ahí; este repo solo distribuye el resultado ya convertido.
