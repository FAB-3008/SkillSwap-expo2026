# SkillSwap — Sistema Combinado

Fusión de **dos versiones de la misma aplicación** en una sola:

- **Base**: el sistema que veníamos trabajando (el de `/var/www/html/skillswap`), el más avanzado:
  certificados con evidencia, banco de horas asimétrico, negociación de horas, rate-limit de login, CSRF, etc.
- **Aportado**: la versión que trabajó otra persona (`~/Descargas/SkillSwap-main`), de la que se tomaron
  las funciones que faltaban: notificaciones, chat con polling, modalidad de sesión, insignias,
  avatar, constancia de horas imprimible, gráficas y exportación CSV.

Los dos orígenes quedaron **intactos** y respaldados en `~/Descargas/respaldo-skillswap-origen/`.

---

## 1. Estructura

```
skillswap-combinado/
├── index.php                      -> redirige a /skillswap/
├── README.md                      -> este documento
├── pruebas/
│   └── combinado_e2e.php          -> 99 pruebas automáticas (requiere el sistema desplegado)
└── skillswap/                     -> la aplicación (se copia como /var/www/html/skillswap)
    ├── admin/       (5 archivos)  -> index, users, categories, reports, export (CSV)
    ├── api/         (2 archivos)  -> notifications.php, messages.php (JSON + CSRF)
    ├── assets/      -> css, js, img/avatars/
    ├── config/      -> db.example.php (plantilla) · db.php (real, NO versionado)
    ├── database/    -> skillswap.sql (12 tablas, listo para instalar)
    ├── includes/    -> auth, functions, header, footer, certificate (PDF) y dompdf/ (librería)
    ├── modules/     -> auth, dashboard, exchanges, matches, messages, profile, ratings, skills
    ├── uploads/     -> certificados adjuntos
    └── .htaccess    -> endurecimiento para hosting compartido (AllowOverride All)
```

## 2. Qué se agregó (y de dónde vino)

| Funcionalidad | Origen | Dónde quedó en el combinado |
|---|---|---|
| Notificaciones (campana, dropdown, marcar leídas) | SkillSwap-main | `includes/functions.php`, `includes/header.php`, `api/notifications.php` |
| Avisos automáticos: propuesta, aceptar, rechazar, cancelar, confirmar, completar, valorar, mensaje | SkillSwap-main (parcial) | `propose.php`, `accept.php`, `complete.php`, `ratings/submit.php`, `api/messages.php` |
| Chat asíncrono con polling (4 s) y envío sin recargar | SkillSwap-main | `api/messages.php`, `modules/messages/index.php` |
| Modalidad de sesión virtual/presencial (enlace o punto de encuentro) | SkillSwap-main | `propose.php`, `modules/exchanges/index.php`, `admin/reports.php` |
| Insignias (Pionero, Mentor Experimentado, Tutor 5 Estrellas, Máster del Tiempo) | SkillSwap-main | `getUserBadges()`, perfiles propio y público |
| Foto de perfil (avatar) | SkillSwap-main (corregido) | `uploadAvatarImage()`, `modules/profile/edit.php` |
| Biografía + enlaces GitHub/LinkedIn | SkillSwap-main | `modules/profile/edit.php`, perfiles |
| Constancia de horas en **PDF real** (A4 fija, una sola hoja) | SkillSwap-main (reescrita) | `modules/profile/certificate.php` + `includes/certificate.php` |
| Gráficas Chart.js en reportes | SkillSwap-main | `admin/reports.php` |
| Exportación CSV de usuarios e intercambios | SkillSwap-main (corregido) | `admin/export.php` |

## 3. Lo que se conservó del sistema base

- Certificados con evidencia (URL + archivo, skill asociada) y gatekeeper `modules/profile/file.php`.
- Banco de horas asimétrico (`proposer_hours` / `acceptor_hours`, decimal) con saldo nunca negativo.
- Negociación de horas (`extend.php`), fecha programada, deduplicación de propuestas.
- Confirmación doble de finalización con transacción y `FOR UPDATE`.
- Rate-limit de login (`login_attempts`), CSRF en todos los formularios y APIs.
- Validación de archivos por contenido, política de privacidad, UI con temas claro/oscuro.

## 4. Correcciones aplicadas al portar

| Problema | Qué se hizo aquí |
|---|---|
| El dump ajeno no incluía `notifications`, `users.bio/github/linkedin` ni `exchanges.meeting_*` | Se regeneró el esquema con las **12 tablas coherentes** en `database/skillswap.sql` |
| Avatar: usaba la extensión enviada por el cliente → un `.php` con contenido JPEG podía quedar ejecutable | `uploadAvatarImage()` valida con `getimagesize()` y **deriva la extensión del contenido** |
| `admin/export.php` consultaba una columna inexistente | CSV reescrito con las columnas reales |
| `api/messages.php` sin validación CSRF | Exige token (formulario o cabecera) |
| Avisos de `notifications` sin manejar errores | `createNotification()` captura excepciones |

**Bug real corregido**: `propose.php`, `messages/index.php` y `ratings/submit.php` imprimían HTML
antes de procesar el POST, así que `redirect()` respondía 200 en vez de 302. Ahora el POST se
procesa siempre antes de emitir la página.

## 5. Esquema de base de datos (12 tablas)

`categories`, `users`, `skills`, `user_skills_offered`, `user_skills_wanted`, `exchanges`,
`ratings`, `hours_log`, `messages`, `notifications`, `certificates`, `login_attempts`.

## 6. Seguridad (endurecimiento aplicado)

**En el código**

- Consultas **preparadas** en todo el proyecto (sin SQLi), **CSRF** en todos los POST y APIs.
- Contraseñas con `password_hash`/`password_verify`; contraseña temporal de reseteo aleatoria.
- Subidas validadas por **contenido** (PNG/JPG por `getimagesize`, PDF por firma `%PDF`), tamaño
  máximo y nombre aleatorio; **la extensión se deriva del contenido**.
- Sesiones: `httponly`, `SameSite=Lax`, `Secure` bajo HTTPS y **modo estricto** + `session_regenerate_id` al autenticar.
- Cabeceras de seguridad: `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`,
  `Referrer-Policy`, `Permissions-Policy`, `Cross-Origin-Opener/Resource-Policy` y **HSTS** en dominios reales.
- **Anti fuerza bruta**: 5 fallos por correo+IP y 25 por IP en 15 min. **Anti-bot en el registro**:
  campo trampa (honeypot) + límite de 10 registros por IP y hora. Poda automática de `login_attempts`.
- Salidas escapadas (`htmlspecialchars`), incluidos los iconos que administra el panel.

**En el servidor** (script `~/Descargas/preparado-https/preparar-servidor.sh`, se ejecuta con `sudo`)

- **HTTPS** con redirección 301 desde HTTP para `/skillswap` (certificado autofirmado local).
- **MySQL**: `root` pasa a `auth_socket` (se administra con `sudo mysql`) y el usuario `admin`
  recibe una clave fuerte. La aplicación usa un usuario de **privilegios mínimos** (`skillswap_app`,
  solo `SELECT/INSERT/UPDATE/DELETE` sobre `skillswap`).
- Carpetas de subida propiedad de `www-data` (sin `0777`); `config/db.php` legible solo por Apache.
- Carpetas internas (`database/`, `includes/`, `config/`, `pruebas/`) y ficheros sensibles
  (`.sql`, `.bak`, `.zip`, `.md`, ocultos…) **bloqueados por Apache** y por `.htaccess`.
- **phpMyAdmin deshabilitado** del acceso web (reversible con `sudo a2enconf phpmyadmin`).

## 7. Despliegue

### Opción A — instalar de cero

```bash
# 1) Base de datos (crea el esquema y los datos semilla: 7 categorías, 16 habilidades y la cuenta admin)
sudo mysql < skillswap/database/skillswap.sql

# 2) Usuario de la aplicación con privilegios mínimos
sudo mysql -e "CREATE USER 'skillswap_app'@'localhost' IDENTIFIED BY 'TU_CLAVE';
               GRANT SELECT,INSERT,UPDATE,DELETE ON skillswap.* TO 'skillswap_app'@'localhost'; FLUSH PRIVILEGES;"

# 3) Código
sudo cp -a skillswap /var/www/html/
cp skillswap/config/db.example.php /var/www/html/skillswap/config/db.php   # y editar usuario/clave

# 4) Permisos + HTTPS + endurecimiento
sudo bash ~/Descargas/preparado-https/preparar-servidor.sh
```

El dump trae los **12 objetos de esquema** y datos semilla; la cuenta `admin@skillswap.com` viene
con una contraseña fuerte. **Cámbiala** tras el primer ingreso (Mi perfil). La contraseña local de
esta instalación está en `~/Descargas/preparado-https/credenciales-admin.txt` (no versionada).

### Opción B — actualizar una instalación existente

```bash
mysqldump -u skillswap_app -p skillswap > respaldo.sql      # respaldar primero
# copiar encima los archivos PHP/CSS/JS de skillswap-combinado/skillswap/
```

### Verificación

```bash
php ~/Descargas/skillswap-combinado/pruebas/combinado_e2e.php    # esperado: 74/74
```

### Dompdf (constancia en PDF)

La constancia se genera con **Dompdf 3.1.6** (licencia **LGPL-2.1**), incluida en
`skillswap/includes/dompdf/` (~11 MB, 319 archivos, sin necesidad de Composer).
Requiere PHP >= 8.1 y las extensiones `dom`, `mbstring`, `gd` y `zlib` (estándar en
PHP 8.x). La carpeta `includes/` ya está bloqueada por HTTP en el `.htaccess`.

La hoja es **A4 fija** (210 mm x 297 mm) maquetada con tablas (sin flexbox/grid): así
el texto no se desborda ni cambia de forma. En pantalla la hoja se **escala** en
dispositivos angostos en lugar de refluír, y el PDF (`certificate.php?pdf=1`) sale
siempre en **una sola página** (las habilidades se listan hasta 20 y el resto se
resume con "… y N habilidades más").

## 8. Respaldos

- `~/Descargas/respaldo-preproduccion/skillswap-web-*.tar.gz` — árbol desplegado antes del endurecimiento.
- `~/Descargas/respaldo-preproduccion/skillswap-COMPLETA-*.sql` — volcado íntegro previo.
- `~/Descargas/respaldo-preproduccion/basura-webroot/` — archivos retirados del webroot
  (`skillswap.tar.gz`, `prestamos-herramientas.zip`, copia duplicada `skillswap-combinado/`, avatar de prueba).
- `~/Descargas/respaldo-preproduccion/migraciones-redundantes/` — `migrate_v2/v3/combinado.sql` (ya incluidas en el dump).
- `~/Descargas/respaldo-skillswap-origen/` — copia intacta del sistema original.

## 9. Verificación realizada

`pruebas/combinado_e2e.php` — **74/74 correctas** sobre el sistema desplegado en `/skillswap`. Cubre:

- registro, login, permisos por rol y protección del panel admin y del CSV sin sesión;
- propuesta con modalidad virtual/enlace y presencial/lugar, deduplicación;
- la otra persona no puede aceptar, el proponente no puede cancelar tras aceptar;
- doble confirmación, saldos, 4 movimientos de historial y limpieza de saldos;
- notificaciones y mensajes (envío, polling, rechazo con CSRF inválido y de usuarios ajenos);
- avatar: extensión derivada del contenido, `.php` del cliente no se guarda, payload PHP rechazado;
- insignias, perfil público, constancia de horas;
- reportes con las 3 gráficas y CSV de usuarios/intercambios con BOM UTF-8;
- todas las páginas sin errores fatales, avisos ni warnings.

## 10. Limitaciones conocidas

- El certificado local es **autofirmado**: el navegador pedirá aceptar una excepción una vez.
  En un dominio real, reemplazar por un certificado válido (Let's Encrypt) y activar HSTS.
- Si borras el avatar de un usuario desde la BD, el archivo queda huérfano en disco (la app lo
  borra al reemplazarlo o al marcar "quitar foto").
- `config/db.php` no se versiona: al clonar el repo hay que crear el archivo desde `db.example.php`.
