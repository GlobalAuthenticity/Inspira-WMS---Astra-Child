# Astra Child — Sistemadi / Inspira B2B

Tema hijo de **Astra** para la plataforma Sistemadi / Inspira (WMS Latam). Añade todo
el flujo de negocio: autenticación propia, tests psicométricos, resultados, cursos
LearnDash, compra de productos en WooCommerce, correos transaccionales y una API REST
para integrarse con WMS.

- **Theme Name:** astra child theme · **Version:** 1.0.0
- **Template:** `astra`
- **Autor:** Global Authenticity LLC — https://sistemadi.com/

## Stack e integraciones

Depende de plugins externos; sin ellos muchas partes no funcionan:

| Integración | Uso en el tema |
|---|---|
| WooCommerce | checkout, carrito, productos de tests y cursos, Mi cuenta |
| LearnDash | cursos, lecciones, progreso, asignación automática |
| Ultimate Member | registro (`um_registration_complete`), reset de contraseña |
| WatuPRO | motor de los tests |
| YITH My Account Page | menú lateral del panel (reemplazado por uno propio) |
| Elementor | maquetación de páginas; el tema inyecta estilos/JS sobre sus widgets |
| Terceros vía CDN | Chart.js, SweetAlert2, Toastr, Selectize, marked |
| HubSpot | script de tracking en el footer |

## Estructura de carpetas

```
astra-child/
├── functions.php              Punto de entrada: requires, hooks y fixes globales
├── style.css                  Cabecera del tema + fixes CSS (Elementor/Woo)
├── header.php                 Override de header con nav propio (menú "Main Header")
├── comments.php
├── functions/                 Bootstrap y hooks transversales
│   ├── enqueue-php-files.php  Carga todos los módulos de app/ (en 'init')
│   ├── enqueue-scripts.php    Encola CSS/JS y expone datos PHP→JS (code_scripts)
│   ├── actions.php            Acciones: carga shortcodes/modales, carrito, registro
│   ├── filters.php            Filtros de campos del checkout de WooCommerce
│   ├── helpers.php            send_json_response(), send_email()
│   └── routes-validations.php Redirecciones por página
├── app/                       Lógica de negocio por módulo (AJAX + queries)
│   ├── constants.php          AppConstants: IDs de productos, páginas y URLs
│   ├── auth/                  login, signup, logout, verify-login, protección de rutas
│   ├── test/                  new-test, save-test
│   ├── results/               get-test-results
│   ├── couples/               match-couple, validate-couple
│   ├── courses/               assign-course-ids-to-users
│   ├── products/              añadir cursos al carrito, validar compra de tests
│   ├── emails/                html/ + functions/ + ajax/ de los 4 correos
│   ├── item-menu/             visibilidad de ítems de menú según test completado
│   ├── queries/user-queries.php  UserQueries (SQL directo)
│   └── yith/my-account-menu.php  Menú lateral del dashboard
├── components/
│   ├── shortcodes/            18 shortcodes (auto-cargados por glob)
│   └── modals/                modal-waiting
├── inc/
│   ├── helpers.php            Procesado de respuestas y cálculo de resultados
│   └── api/post/              Endpoints REST de integración (users, areas)
├── apis/api-user-registration.php  Endpoint ms/v1/create-user
├── hooks/
│   ├── wc_hooks.php           Validez de cupones
│   ├── course_assignment_import.php  Asignación de cursos tras el pago (desactivado)
│   └── generate_secure_token_secret.php (desactivado)
├── assets/css · assets/js     Estilos y scripts propios (+ .min generados)
├── woocommerce/               Overrides de plantillas (checkout, order, myaccount)
├── learndash/ld30/lesson.php  Override de plantilla de lección
└── ultimate-member/           Overrides de emails y password-reset
```

## Carga de código

1. `functions.php` hace `require_once` de: `apis/api-user-registration.php`,
   `hooks/wc_hooks.php`, `inc/helpers.php` y los 5 endpoints de `inc/api/post/`.
2. En `init` (prioridad 0) `sistemadi_resources_init()` incluye
   `functions/enqueue-php-files.php`, que a su vez requiere **todos** los módulos de
   `app/` y los archivos de `functions/`.
3. También en `init`, `functions/actions.php` auto-carga por `glob()` cada `.php` de
   `components/shortcodes/` y `components/modals/`.

> Como los shortcodes se cargan por glob, un archivo duplicado en esa carpeta provoca
> «Cannot redeclare». Varios archivos se protegen con `if ( ! function_exists(...) )`.

## `AppConstants` (app/constants.php)

Centraliza los IDs que el negocio usa en varios sitios:

- Tipos de test: `padres`, `pareja`, `personal`.
- `TESTS_PRODUCTS_IDS` → producto WooCommerce de cada test (1118 / 1382 / 797).
- `PAGE_RESULT_*` → clases `page-id-*` de las páginas de resultados.
- `TEST_RESULT_URL_BY_TYPE` → slug de resultados por tipo.
- Catálogos de cursos: estrategias (7), amenazas (8), oportunidades (4), módulo de
  matrícula, y los bundles «todos los cursos de…».
- Helpers: `getCoursesProductsIds()`, `getCoursesBundlesIds()`,
  `urlComprarTestByType()`, `urlComprarTestById()`.

## Endpoints AJAX (`admin-ajax.php`)

Todos registrados en pares `wp_ajax_` / `wp_ajax_nopriv_`:

| Acción | Módulo |
|---|---|
| `custom_login`, `custom_signup_user`, `custom_logout`, `is_user_logged_in` | auth |
| `prepare_new_test`, `save_test_answers` | test |
| `get_test_results` | results |
| `match_couple`, `validate_couple` | couples |
| `get_assign_course_ids_to_users` | courses |
| `ids_wc_add_product_to_cart`, `ids_wc_add_category_all_to_cart`, `validate_user_bought_tests` | products |
| `ajax_send_couple_email` | emails |
| `get_item_view_menu` | item-menu |

Los nonces que consumen se generan en `enqueue-scripts.php` y viajan al front en el
objeto JS global **`code_scripts`** (junto con URLs del sitio, IDs de producto y el
nombre del usuario).

## API REST

| Método | Ruta | Autenticación | Archivo |
|---|---|---|---|
| GET | `custom/v1/user?email=` | header `x-api-key` contra `CUSTOM_API_KEY` | `functions.php` |
| POST | `ms/v1/create-user` | token | `apis/api-user-registration.php` |
| POST | `aprobandome/v1/create-user` | token (`aprobandome_create_admin_user`) | `inc/api/post/user/create-user-admin-post.php` |
| POST | `aprobandome/v1/create-user-member` | token (`aprobandome_create_user_member`) | `inc/api/post/user/create-user-member-post.php` |
| POST | `aprobandome/v1/user-id-by-email` | token (`aprobandome_user_id_by_email`) | `inc/api/post/user/user-id-by-email.php` |
| POST | `aprobandome/v1/areas` | token (`aprobandome_create_area`) | `inc/api/post/area/create-area-post.php` |
| POST | `aprobandome/v1/actualizar-area` | token (`aprobandome_update_area`) | `inc/api/post/area/update-area-post.php` |

Los tokens `aprobandome_*` se siembran en `init` desde `functions.php` como opciones de
WordPress con un valor por defecto. La creación de admin asigna el rol
`empresa_admin` y genera un `mi_autologin_token` con 15 minutos de vigencia.

**Nota de seguridad:** `CUSTOM_API_KEY` y el token predefinido `aprobandome_*` están
escritos en claro en `functions.php`. Conviene moverlos a `wp-config.php` o a variables
de entorno antes de tocar producción.

## Shortcodes

| Shortcode | Qué hace |
|---|---|
| `[sistemadi_login]` · `[sistemadi_signup]` | formularios de acceso propios |
| `[custom_forgot_password]` · `[custom_reset_password]` | recuperar / fijar contraseña |
| `[apro_test_steps]` | barra de pasos del test |
| `[pending_test]` · `[couple_pending_test]` | snackbars de test pendiente |
| `[validate_couple_code]` | validación del código de pareja |
| `[personal_result_review]` | resumen del resultado individual |
| `[report-content-table]` · `[couples-report-content-table]` | tabla de contenidos del informe |
| `[process_steps]` | pasos del proceso |
| `[ids-cursos-learndash]` | listado de cursos LearnDash del usuario |
| `[user-general-progress]` | progreso general de cursos (solo `subscriber`) |
| `[view-user-membership-details]` | detalles de la membresía |
| `[group_detail]` | panel de grupo/área para `empresa_admin` |
| `[asistente_virtual_chat]` | chat «Efrén IA» contra un webhook externo |

## Roles y reglas de acceso

- **`subscriber`** — usuario final. Requisito para ver páginas de resultados.
- **`empresa_admin`** — administrador de empresa/área. Ve `[group_detail]` y accede a
  Efrén IA sin haber hecho el test.

`app/auth/routes-protection.php` aplica, en `template_redirect`:

- Páginas de resultados: exige rol `subscriber` **y** el usermeta
  `question_test_ids` (prueba de test completado); si falta, redirige al home.
- `/registro` con sesión abierta → `/test/?test-type=individual`.
- Sin sesión → redirige a `https://wmslatam.com/` (salvo editor de Elementor).
- Con sesión en el home → `/mi-cuenta/panel-usuario/`.

`question_test_ids` es la bandera central de «test completado»; también la usa
`UserQueries::user_completed_test()` y el bloqueo de la tarjeta Efrén IA.

## Flujos principales

**Registro → test → compra.** Ultimate Member dispara `um_registration_complete`, que
envía el correo de bienvenida e inicia sesión redirigiendo a `/test/?test-type=…` (o a
`/verificacion-pareja/` si venía con `?pareja2=true`). Si el usuario ya completó el
test y vuelve a `/registro`, se le lleva al checkout del producto correspondiente o al
panel.

**Carrito de tests.** `clean_cart_and_add_product()` vacía el carrito antes de añadir un
producto de test, para que nunca convivan dos tests; usa `?previous=true` como guarda
anti-bucle.

**Test y resultados.** El test se guarda por AJAX (`save_test_answers`); `inc/helpers.php`
procesa las respuestas (`process_answers`, `get_lvl`) y calcula resultados por usuario y
por área (`get_results_user`, `get_results_area`). Los resultados se pintan con Chart.js
desde `assets/js/results-ids.js`.

**Parejas.** Un segundo usuario valida un código, se emparejan (`match_couple` →
`UserQueries::create_couple_match`) y se dispara el correo de pareja.

**Post-pago.** `hooks/course_assignment_import.php` asigna cursos LearnDash al
completarse el pago (`woocommerce_payment_complete`, `order_status_processing`,
`order_status_completed`). **Está comentado en `functions.php`**, así que hoy no se
carga.

**Correos.** Cuatro plantillas HTML con su función de envío: bienvenida, resultados,
pareja y curso.

## Checkout de WooCommerce

- `functions/filters.php` quita campos (empresa, dirección 2, código postal, todo
  shipping, comentarios) y traduce labels/placeholders al español.
- Campo extra `couple_email`: validado en `woocommerce_checkout_process`, guardado como
  meta `_couple_email` y mostrado en el pedido del admin.
- Afiliados: las cookies `affiliate_ref` y `affiliate_ref_aprobandome` se guardan en el
  pedido y se añaden al payload de los webhooks de WooCommerce.
- Redirección post-login fija a `/mi-cuenta/panel-usuario/`; logout → `/cerrar-sesion`.
- Plantillas sobrescritas en `woocommerce/` (checkout, review-order, thankyou,
  order-details, form-edit-account).

## Parches y workarounds en `functions.php`

Conviene conocerlos porque son frágiles y dependen del marcado:

- **`test-type` forzado a `individual`**: redirección PHP de `/test` sin parámetro y un
  script en el footer que reescribe formularios, enlaces y botones.
- **Vídeo LearnDash en Safari**: habilita «Marcar como completado» al 90 % o al acabar
  el vídeo, y desactiva `learndash_video_progression` en Safari.
- **HTTPS forzado** en `script_loader_src`, `style_loader_src` y `content_url`.
- **Tarjeta Efrén IA bloqueada** (`#efrenia`): blur + candado + modal si el usuario no
  tiene `question_test_ids`; `empresa_admin` queda exento.
- **Eliminación del ítem de menú «areas admin»** con un `preg_replace` sobre el HTML
  mediante `ob_start()` en `wp_head`.
- **Anti-bucle de `redirect_to`**: corta en `init` las URLs con `redirect_to` anidado y
  quita en el home el `template_redirect` de Ultimate Member.
- **Logs de depuración activos**: tres `add_action('template_redirect', …)` escriben en
  `error_log` en cada carga del home, y `save_couple_email_metadata()` loguea todo el
  `$_POST`. Deberían retirarse en producción.

## Trabajar sobre el tema

- Los `.min.css` del repo son copias minificadas de los `.css`; al editar el fuente hay
  que regenerar el minificado o el cambio no se verá donde se encole la versión `.min`.
- `enqueue-scripts.php` encola **todos** los CSS y JS en todas las páginas, sin
  condicionales. Al añadir un asset, considera acotarlo por página.
- Cualquier módulo nuevo de `app/` debe añadirse a
  `functions/enqueue-php-files.php`; los shortcodes y modales se cargan solos.
- `astra-child.zip` y `screenshot.jpg` son artefactos de empaquetado, no código.
