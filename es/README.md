# B2C Commerce Cloud — Evaluación de Arquitectura de Storefront

**Autor:** Luciano Diaz — Principal Solution Engineer en Salesforce

**Credenciales Salesforce:**
- Certified B2C Commerce Developer
- Certified Data Cloud Consultant
- Certified Agentforce Specialist
- Certified AI Associate
- Certified Platform Foundations

**Fecha:** Junio 2026

---

## Objetivo del Documento

El presente documento tiene como objetivo realizar un relevamiento técnico-funcional de las alternativas de implementación disponibles para Salesforce B2C Commerce Cloud en el contexto de una evaluación de arquitectura para una organización que busca modernizar o evolucionar su canal de comercio digital. A diferencia de una comparación comercial o de una descripción general de capacidades, este análisis se concentra en explicar las implicancias arquitectónicas, funcionales y operativas de los tres enfoques principales de construcción de storefront sobre B2C Commerce Cloud: Storefront Reference Architecture, PWA Kit / Composable Storefront y una aproximación headless o composable a nivel tecnológico. El propósito es entregar una base clara para comprender cómo cada modalidad resuelve la experiencia digital de comercio, qué nivel de flexibilidad ofrece, qué componentes de Salesforce intervienen, cuáles son sus dependencias técnicas y qué consideraciones deben evaluarse en términos de time-to-market, mantenibilidad, escalabilidad, integración y evolución futura.

Este documento toma como punto de partida la necesidad de comprender, con mayor profundidad, las opciones de implementación que Salesforce B2C Commerce Cloud ofrece para soportar una estrategia de comercio digital moderna. En este sentido, el relevamiento no busca definir únicamente una preferencia tecnológica, sino establecer criterios de decisión que permitan contrastar cada alternativa frente a las necesidades del negocio, la madurez tecnológica del ecosistema actual, la capacidad de integración con sistemas corporativos, la experiencia esperada para los usuarios finales y el modelo operativo requerido para sostener la plataforma en el tiempo. Para cada enfoque se analizará el rol del storefront, la relación con B2C Commerce Cloud, el uso de APIs, la separación entre experiencia de usuario y lógica de comercio, el grado de acoplamiento con la plataforma y los escenarios en los cuales cada alternativa puede resultar más conveniente según el contexto de la organización.

---

## 1. Storefront Reference Architecture (SFRA)

### Qué es SFRA

Storefront Reference Architecture (SFRA) es el storefront de referencia oficial provisto por Salesforce como base para construir storefronts de B2C Commerce Cloud. Es una aplicación renderizada del lado del servidor que se ejecuta completamente dentro de la infraestructura de la plataforma B2C Commerce Cloud, lo que significa que tanto la lógica de comercio como la capa de presentación están fuertemente acopladas y se ejecutan en los servidores de Salesforce.

SFRA reemplazó a la arquitectura de referencia anterior conocida como SiteGenesis, ofreciendo una base de código más modular y mantenible preservando el mismo paradigma fundamental de renderizado del lado del servidor. Sirve tanto como punto de partida funcional para nuevas implementaciones como demostración de las mejores prácticas de la plataforma.

### Modelo Arquitectónico

SFRA sigue un patrón Model-View-Controller (MVC) ejecutado del lado del servidor en el application server de B2C Commerce Cloud:

- **Controllers** son archivos JavaScript del lado del servidor que manejan las solicitudes HTTP entrantes, orquestan la lógica de negocio y preparan los datos para el renderizado. Se ejecutan en el motor de scripts de Salesforce B2C Commerce (basado en Rhino) y tienen acceso directo a las APIs de la plataforma para operaciones de catálogo, carrito, checkout, clientes y órdenes.

- **Models** son objetos JavaScript que encapsulan y transforman datos crudos de la plataforma en representaciones estructuradas consumidas por la capa de vista. Actúan como una abstracción entre el modelo de datos interno de la plataforma y lo que los templates necesitan para renderizar.

- **Views** son templates ISML (Internet Store Markup Language) — un lenguaje de templates propietario de Salesforce que genera HTML en el servidor antes de enviar la respuesta al navegador. ISML soporta loops, condicionales, inclusión de templates y patrones decorator para composición de layouts.

### Sistema de Cartridges y Herencia

El mecanismo central de extensibilidad en SFRA es el **cartridge path** — una lista ordenada de cartridges que determina cómo la plataforma resuelve controllers, templates, assets estáticos y scripts en tiempo de ejecución. Cada cartridge es esencialmente un módulo autocontenido con una estructura de directorios definida.

Cuando llega una solicitud, la plataforma recorre el cartridge path de izquierda a derecha, resolviendo el primer controller o template que coincida. Esto habilita un modelo de override en capas:

- **Cartridges base** (`app_storefront_base`) contienen la implementación por defecto de SFRA.
- **Cartridges custom** se colocan antes en el path y pueden sobreescribir o extender cualquier controller, template o model de la base.
- **Cartridges de plugins** proveen funcionalidad opcional (por ejemplo, `plugin_applepay`, `plugin_wishlist`) y se insertan entre los cartridges custom y los base.

Este modelo de herencia permite a los equipos personalizar el comportamiento sin modificar el código base, lo que simplifica las rutas de actualización cuando Salesforce lanza nuevas versiones de SFRA.

### Renderizado del Lado del Servidor y Ciclo de Vida de una Solicitud

Cada solicitud de página en SFRA sigue este ciclo de vida:

1. El navegador envía una solicitud HTTP a la capa CDN/edge de B2C Commerce Cloud.
2. Si no está en caché, la solicitud llega al application server y se enruta al controller apropiado.
3. El controller ejecuta la lógica de negocio usando las APIs de scripts de la plataforma (accediendo a catálogos, precios, promociones, inventario, etc.).
4. El controller pasa un view model a un template ISML.
5. El template renderiza el HTML final en el servidor.
6. La respuesta HTML completa se devuelve al navegador.

El JavaScript del lado del cliente en SFRA se limita a progressive enhancement — validación de formularios, llamadas AJAX para actualizaciones del mini-carrito e interacciones de UI. La estructura y el contenido de la página siempre se resuelven del lado del servidor.

### Relación con Business Manager

SFRA está profundamente integrado con Business Manager, la aplicación de back-office de B2C Commerce Cloud. Los merchants usan Business Manager para gestionar:

- Catálogos de productos, categorías y precios
- Promociones y campañas
- Content slots y layouts de página
- Preferencias y configuración del sitio
- Segmentos de clientes y A/B testing

Los cambios realizados en Business Manager se reflejan inmediatamente en el storefront de SFRA sin requerir despliegues de código. Este acoplamiento fuerte entre la herramienta de merchandising y el storefront es una característica definitoria del modelo SFRA.

### Page Designer

Page Designer es el editor visual de páginas no-code integrado en Business Manager. Permite a los merchants crear, organizar y publicar páginas arrastrando y soltando componentes en regiones — sin escribir código ni requerir despliegues.

En el contexto de SFRA, Page Designer trabaja con tipos de componentes basados en ISML. Los desarrolladores construyen tipos de página y tipos de componentes reutilizables (banners, carruseles de productos, hero images, bloques de contenido) usando templates ISML, y los merchants los ensamblan visualmente en Business Manager. La arquitectura sigue una jerarquía de tres niveles:

- **Pages** — contenedores de nivel superior que representan una página completa (homepage, landing page, página de campaña).
- **Regions** — áreas de contenido lógicas dentro de una página que contienen una lista ordenada de componentes.
- **Components** — los bloques de construcción. Los componentes leaf renderizan contenido directamente (imágenes, texto, tiles de producto). Los componentes de layout organizan otros componentes usando grillas o columnas y pueden contener regiones anidadas.

Los merchants pueden programar la publicación de páginas, previsualizar cambios en contexto y gestionar múltiples versiones de página — todo desde Business Manager sin intervención de desarrolladores para actualizaciones de contenido.

### Flujo de Desarrollo

El desarrollo en SFRA se apoya en las siguientes herramientas y procesos:

- **Prophet Debugger / extensiones B2C Commerce para IDE** en VS Code proveen upload de código, debugging y streaming de logs directamente contra instancias sandbox.
- **El despliegue de código** se realiza subiendo cartridges a un sandbox vía WebDAV o a través de pipelines CI/CD usando el CLI de Salesforce B2C Commerce (sfcc-ci).
- **Sandboxes** son ambientes de desarrollo individuales provistos por la plataforma, cada uno ejecutando una instancia completa de B2C Commerce Cloud.
- **El linting y compilación de ISML** ocurre al momento del upload — no hay un paso de build local para templates.

El ciclo de feedback del desarrollo implica subir código a un sandbox remoto y testear contra ese ambiente, ya que no existe un runtime local que replique el motor de scripts de B2C Commerce.

### Resumen del Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| Runtime | Application server de B2C Commerce Cloud (motor JS basado en Rhino) |
| Controllers | JavaScript del lado del servidor (módulos CommonJS) |
| Templates | ISML (Internet Store Markup Language) |
| Lado del cliente | jQuery, JavaScript vanilla, AJAX |
| Estilos | SCSS compilado a CSS (via build tools) |
| Build tools | Webpack (solo para assets del lado del cliente) |
| Despliegue | Upload WebDAV / sfcc-ci CLI |
| Back-office | Business Manager |
| APIs consumidas | Platform Script API (del lado del servidor, interna) |

### Ejemplos de Código

#### Template ISML — Layout de Página con Inclusión de CSS y JS

```isml
<isdecorate template="common/layout/page">

    <isscript>
        var assets = require('*/cartridge/scripts/assets');
        assets.addCss('/css/product/detail.css');
        assets.addJs('/js/product/detail.js');
    </isscript>

    <div class="container product-detail" data-pid="${product.id}">
        <div class="row">
            <div class="col-12 col-sm-6">
                <isinclude template="product/components/imageCarousel" />
            </div>
            <div class="col-12 col-sm-6">
                <h1 class="product-name">${product.productName}</h1>
                <div class="prices">
                    <isset name="price" value="${product.price}" scope="page" />
                    <isinclude template="product/components/pricing/main" />
                </div>
                <isif condition="${product.available}">
                    <button class="add-to-cart btn btn-primary">
                        ${Resource.msg('button.addtocart', 'common', null)}
                    </button>
                <iselse/>
                    <span class="out-of-stock">
                        ${Resource.msg('label.outofstock', 'common', null)}
                    </span>
                </isif>
            </div>
        </div>
    </div>

</isdecorate>
```

#### Controller — JavaScript del Lado del Servidor

```javascript
'use strict';

var server = require('server');
var cache = require('*/cartridge/scripts/middleware/cache');
var ProductFactory = require('*/cartridge/scripts/factories/product');

server.get('Show', cache.applyDefaultCache, function (req, res, next) {
    var params = req.querystring;
    var product = ProductFactory.get({ pid: params.pid });

    res.render('product/productDetails', {
        product: product
    });

    next();
});

module.exports = server.exports();
```

#### JavaScript del Lado del Cliente — AJAX Add to Cart

```javascript
'use strict';

var processInclude = require('base/util');

$(document).ready(function () {
    $('.add-to-cart').on('click', function (e) {
        e.preventDefault();
        var pid = $(this).closest('.product-detail').data('pid');

        $.ajax({
            url: '/on/demandware.store/Sites-SiteId-Site/default/Cart-AddProduct',
            method: 'POST',
            data: { pid: pid, quantity: 1 },
            success: function (data) {
                $('.minicart-quantity').text(data.quantityTotal);
                $('.minicart').trigger('count:update', data);
            }
        });
    });
});
```

### Cuándo Usar SFRA

SFRA es el enfoque estándar para lanzar un storefront en B2C Commerce Cloud. Trabaja con JavaScript nativo y el ecosistema propio de la plataforma (ISML, cartridges, Business Manager) sin depender de librerías frontend externas como React o Vue. Representa el primer grado de implementación — el camino directo y completo, con todo integrado.

### Fortalezas

- Solución integral out-of-the-box con funcionalidad de comercio completa desde el día uno.
- Capacidades de merchandising inmediatas a través de Business Manager — no se necesitan despliegues de código para cambios de contenido, promociones o catálogo.
- Menor complejidad operativa — no hay infraestructura de frontend separada que hostear, escalar o monitorear.
- Maduro y probado a escala, con sitios espectaculares en producción en diversas industrias a nivel mundial.

### Consideraciones

- El frontend está acoplado a la plataforma, lo que simplifica las operaciones pero implica que el desarrollo de UX se resuelve dentro del ecosistema de B2C Commerce en lugar de con librerías externas como React o Vue.
- El desarrollo requiere subir código a sandboxes remotos — no hay runtime local para el motor de scripts de B2C Commerce.
- El modelo de renderizado del lado del servidor implica que las transiciones de página involucran round-trips completos al servidor, a diferencia de las arquitecturas de single-page application.

### Documentación Oficial

| Recurso | URL |
|---------|-----|
| B2C Commerce Cloud Developer Docs | https://developer.salesforce.com/docs/commerce/b2c-commerce/overview |
| SFRA Overview | https://developer.salesforce.com/docs/commerce/sfra/overview |
| SFRA Architecture Guide | https://developer.salesforce.com/docs/commerce/sfra/guide/sfra-overview.html |
| Cartridges | https://developer.salesforce.com/docs/commerce/b2c-commerce/guide/b2c-cartridges.html |
| ISML Templates | https://developer.salesforce.com/docs/commerce/b2c-commerce/guide/b2c-isml.html |
| Script API Reference (Developer Doc) | https://salesforcecommercecloud.github.io/b2c-dev-doc/ |
| Commerce APIs (SCAPI) | https://developer.salesforce.com/docs/commerce/commerce-api/references/ |
| SFRA GitHub Repository | https://github.com/SalesforceCommerceCloud/storefront-reference-architecture |

---

## 2. PWA Kit / Composable Storefront

### Qué es Composable Storefront

Composable Storefront es la solución de storefront desacoplado de Salesforce para B2C Commerce Cloud. Separa el frontend (capa de presentación) del backend de comercio, conectándolos exclusivamente a través de APIs. El término "composable" se refiere al enfoque arquitectónico donde el storefront se ensambla a partir de componentes independientes conectados por API en lugar de ejecutarse como una aplicación monolítica en el servidor de comercio.

A diferencia de SFRA, donde los templates y la lógica se ejecutan en el application server de B2C Commerce, Composable Storefront corre en su propia infraestructura dedicada (Managed Runtime) y consume los servicios de B2C Commerce a través de la Shopper Commerce API (SCAPI). El motor de comercio sigue siendo el mismo — catálogos, precios, promociones, checkout — pero la forma en que el storefront accede y presenta esos datos cambia fundamentalmente.

Salesforce empaqueta esta solución en dos componentes principales:

- **PWA Kit** — el framework de desarrollo (basado en React) para construir la aplicación del storefront.
- **Managed Runtime** — la infraestructura de hosting y despliegue provista por Salesforce para ejecutar el storefront a escala.

### Qué es PWA Kit

PWA Kit es un framework de desarrollo open-source construido sobre React que provee las herramientas, librerías y estructura de proyecto necesarias para construir un storefront de B2C Commerce como una Progressive Web Application. Está organizado como un monorepo que contiene:

- **Retail React App** — una implementación completa de storefront de referencia con páginas para listado de productos, detalle de producto, carrito, checkout, gestión de cuenta y store locator. Sirve tanto como punto de partida funcional como demostración de mejores prácticas.
- **commerce-sdk-react** — una librería de React hooks para interactuar con las APIs de B2C Commerce (SCAPI). Provee acceso tipado y declarativo a productos, categorías, baskets, órdenes, promociones y datos de clientes.
- **pwa-kit-dev** — herramientas de desarrollo para desarrollo local, building y despliegue de bundles.
- **pwa-kit-runtime** — el runtime del lado del servidor que maneja SSR (Server-Side Rendering) y routing de solicitudes en producción.

La Retail React App usa Chakra UI como librería de componentes, proporcionando más de 50 componentes de UI accesibles con un sistema de temas basado en la Styled System Theme Specification. Los estilos son completamente personalizables a través de theme tokens.

### Modelo Arquitectónico

Composable Storefront sigue una arquitectura desacoplada donde el frontend y el backend son sistemas independientes conectados por APIs:

```
┌─────────────────────────────────────────────────────────┐
│                      Navegador                           │
│  (React SPA con navegación del lado del cliente)        │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                  Managed Runtime                          │
│  ┌─────────┐  ┌──────────┐  ┌────────────────────┐     │
│  │   CDN   │  │   WAF    │  │  Node.js / Express │     │
│  │ (Edge)  │  │          │  │  (SSR + Routing)   │     │
│  └─────────┘  └──────────┘  └────────────────────┘     │
└──────────────────────────┬──────────────────────────────┘
                           │ SCAPI (REST)
┌──────────────────────────▼──────────────────────────────┐
│              B2C Commerce Cloud                           │
│  (Catálogo, Precios, Promociones, Carrito, Checkout)    │
└─────────────────────────────────────────────────────────┘
```

- **Server-Side Rendering (SSR)** — la carga inicial de página se renderiza en el servidor Node.js dentro de Managed Runtime, entregando una página HTML completamente formada para performance y SEO. Después de la hidratación, la app transiciona a una single-page application (SPA) del lado del cliente con navegación instantánea entre páginas.
- **API-first** — todos los datos de comercio se obtienen via endpoints REST de SCAPI. No hay acceso directo al motor de scripts de B2C Commerce ni a las APIs internas de la plataforma desde el código del storefront.
- **Capa proxy** — Managed Runtime incluye un servidor proxy que acelera las solicitudes API entre el storefront y B2C Commerce, reduciendo la latencia al mantener conexiones activas y rutear eficientemente.

### Managed Runtime

Managed Runtime es la infraestructura de hosting que Salesforce provee específicamente para ejecutar aplicaciones Composable Storefront. No es un servicio de cloud hosting genérico — está construido específicamente para proyectos PWA Kit.

Características clave:

- **CDN con caching en el edge** — las solicitudes se cachean en el edge, cerca de los usuarios finales, para una entrega rápida de páginas.
- **Web Application Firewall (WAF)** — protege los ambientes de amenazas de seguridad automáticamente.
- **Microservicios con auto-scaling** — los ambientes escalan hacia arriba y abajo según el tráfico sin intervención manual.
- **Múltiples ambientes** — los equipos pueden crear ambientes ilimitados (desarrollo, staging, QA, producción) sin costo adicional. Los ambientes son livianos y se pueden provisionar o eliminar a demanda.
- **Despliegue regional** — los admins pueden configurar en qué regiones se despliega el storefront, optimizando la latencia para las audiencias objetivo.
- **Despliegue basado en bundles** — el código se empaqueta en bundles (máximo 400 MB) y se despliega via Runtime Admin o la API de Managed Runtime. Los despliegues iniciales toman hasta una hora; los despliegues subsiguientes se completan en aproximadamente un minuto.

### Relación con B2C Commerce Cloud

El backend de comercio sigue siendo B2C Commerce Cloud — el mismo motor que potencia SFRA. Los merchants siguen usando Business Manager para gestionar catálogos, precios, promociones y campañas. La diferencia está en cómo el storefront accede a estos datos:

| Aspecto | SFRA | Composable Storefront |
|---------|------|----------------------|
| Acceso a datos | Platform Script API (interna, del lado del servidor) | SCAPI (API REST externa) |
| Renderizado | Del lado del servidor en el app server de B2C Commerce | SSR en Managed Runtime + SPA del lado del cliente |
| Hosting | Infraestructura de B2C Commerce | Managed Runtime (separado) |
| Merchandising | Business Manager → reflejo inmediato | Business Manager → expuesto via SCAPI |

La funcionalidad de Business Manager se preserva — los merchants gestionan los mismos datos de comercio. El storefront simplemente los consume de forma diferente.

### Modelo de Autenticación

Composable Storefront usa **SLAS (Shopper Login and API Access Service)** para autenticación. SLAS provee:

- Gestión de tokens para shoppers guest y registrados
- Flujos OAuth 2.0 para acceso seguro a APIs
- Session bridging entre el storefront y B2C Commerce

Es una capa de autenticación separada de lo que SFRA usa internamente, diseñada específicamente para patrones de acceso API-first.

### Page Designer

Page Designer se integra con Composable Storefront a través de un modelo de renderizado headless. Los tipos de componentes se construyen como componentes React que renderizan dentro de la aplicación PWA Kit. Los merchants usan el mismo editor visual de drag-and-drop en Business Manager para crear y organizar páginas.

Cómo funciona:

- Todos los archivos de metadata de Page Designer (pages, components, aspect types) deben incluir `"arch_type": "headless"` para habilitar el renderizado React.
- Los componentes se cargan bajo demanda (lazy-loaded) a través de un sistema de registro de componentes, lo que reduce el tamaño inicial del bundle — solo los componentes usados en una página determinada se descargan.
- El editor visual en Business Manager carga el storefront PWA Kit real en un iframe, así los merchants previsualizan exactamente cómo se verá la página en vivo en la ruta real.
- Los desarrolladores construyen tipos de componentes como componentes React que reciben su configuración como props desde Page Designer.

La arquitectura sigue una jerarquía de tres niveles: Pages (contenedores de nivel superior con definiciones de ruta) → Regions (áreas de contenido que contienen listas ordenadas de componentes) → Components (componentes leaf para renderizado de contenido, componentes de layout para organizar regiones anidadas).

Los merchants pueden programar la publicación de páginas, previsualizar en contexto y gestionar versiones — todo desde Business Manager sin despliegues de código.

Requisitos: PWA Kit versión 2.7.0 o posterior, y un cliente SLAS configurado con el scope `sfcc.shopper-experience`.

### Integraciones de Búsqueda y Descubrimiento

Dentro del ecosistema de Composable Storefront, Salesforce soporta integraciones con proveedores especializados de búsqueda:

- **Algolia** — búsqueda en tiempo real, faceting y merchandising
- **Coveo** — búsqueda y recomendaciones potenciadas por IA
- **Constructor** — descubrimiento de productos y personalización

Estas integraciones funcionan junto con las capacidades de búsqueda nativas de B2C Commerce, dando a los equipos la opción de mejorar el descubrimiento de productos sin salir del ecosistema de Composable Storefront.

### Flujo de Desarrollo

La experiencia de desarrollo con PWA Kit es fundamentalmente diferente a SFRA:

- **Desarrollo local** — los desarrolladores ejecutan la aplicación completa localmente en su máquina con hot module replacement. No se necesitan uploads a sandboxes remotos para iterar en el frontend.
- **Runtime Node.js** — la app corre en un servidor estándar Node.js/Express, tanto localmente como en producción.
- **Ecosistema npm** — acceso completo al registro de paquetes npm para librerías y herramientas de terceros.
- **Soporte TypeScript** — TypeScript opcional para type safety en toda la base de código.
- **Testing** — Jest y React Testing Library para testing unitario y de integración, Lighthouse para monitoreo de performance.
- **Despliegue** — los bundles se pushean a Managed Runtime via CLI (`pwa-kit-dev push`) y se despliegan a través de Runtime Admin o API.

El ciclo de feedback es rápido: codear localmente, ver cambios al instante, testear con datos reales de API via SCAPI, y luego pushear y desplegar cuando esté listo.

### Resumen del Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| Runtime | Node.js / Express (en Managed Runtime) |
| Framework | React |
| Lenguaje | JavaScript / TypeScript |
| Librería UI | Chakra UI |
| Routing | React Router |
| Cliente API | commerce-sdk-react (hooks SCAPI) |
| SSR | Server-side rendering con hidratación |
| Estilos | Sistema de temas Chakra UI (Styled System) |
| Build tools | Webpack, Babel |
| Testing | Jest, React Testing Library, Lighthouse |
| Despliegue | pwa-kit-dev CLI → Managed Runtime |
| Hosting | Managed Runtime (CDN + WAF + auto-scaling) |
| Autenticación | SLAS (Shopper Login and API Access Service) |
| APIs consumidas | SCAPI (Shopper Commerce API — REST externa) |

### Ejemplos de Código

#### Componente React — Página de Detalle de Producto

```jsx
import React from 'react'
import { useProduct } from '@salesforce/commerce-sdk-react'
import { Box, Heading, Text, Button, Stack, Image } from '@chakra-ui/react'

const ProductDetail = ({ productId }) => {
    const { data: product, isLoading } = useProduct({ parameters: { id: productId } })

    if (isLoading) return <Box>Cargando...</Box>

    return (
        <Box maxW="container.lg" mx="auto" p={6}>
            <Stack direction={['column', 'row']} spacing={8}>
                <Image
                    src={product?.imageGroups?.[0]?.images?.[0]?.link}
                    alt={product?.name}
                    boxSize="400px"
                    objectFit="cover"
                />
                <Stack spacing={4}>
                    <Heading as="h1" size="lg">
                        {product?.name}
                    </Heading>
                    <Text fontSize="xl" fontWeight="bold">
                        ${product?.price}
                    </Text>
                    <Text>{product?.shortDescription}</Text>
                    <Button colorScheme="blue" size="lg">
                        Agregar al Carrito
                    </Button>
                </Stack>
            </Stack>
        </Box>
    )
}

export default ProductDetail
```

#### Commerce SDK React — Hook de Agregar al Carrito

```jsx
import { useShopperBasketsMutation } from '@salesforce/commerce-sdk-react'

const AddToCartButton = ({ productId, quantity = 1 }) => {
    const addItemToBasket = useShopperBasketsMutation('addItemToBasket')

    const handleAddToCart = async () => {
        await addItemToBasket.mutate({
            parameters: { basketId: currentBasketId },
            body: [{ productId, quantity }]
        })
    }

    return (
        <Button
            onClick={handleAddToCart}
            isLoading={addItemToBasket.isLoading}
            colorScheme="blue"
        >
            Agregar al Carrito
        </Button>
    )
}
```

#### Configuración de Rutas

```jsx
import { RouteConfig } from '@salesforce/pwa-kit-runtime/ssr/universal/components/_app-config'

const routes = [
    {
        path: '/',
        component: Home,
        exact: true
    },
    {
        path: '/category/:categoryId',
        component: ProductList
    },
    {
        path: '/product/:productId',
        component: ProductDetail
    },
    {
        path: '/cart',
        component: Cart
    },
    {
        path: '/checkout',
        component: Checkout
    }
]

export default routes
```

### Cuándo Usar Composable Storefront

Composable Storefront es el segundo grado de implementación en B2C Commerce Cloud. Es el camino para equipos que quieren control total sobre la experiencia del frontend usando tecnologías web modernas (React, Node.js) mientras permanecen dentro del ecosistema managed de Salesforce para hosting y despliegue. El motor de comercio sigue siendo el mismo — la diferencia está en cómo construís y entregás la experiencia del cliente.

### Fortalezas

- Control total sobre el frontend con React y el ecosistema moderno de JavaScript.
- Desarrollo local con hot reload — sin dependencia de sandboxes remotos para trabajo de frontend.
- Capacidades de Progressive Web App (soporte offline, instalable, experiencia tipo app).
- Server-side rendering para SEO y performance, con navegación SPA del lado del cliente para transiciones de página instantáneas.
- Managed Runtime elimina las preocupaciones de infraestructura — CDN, WAF, scaling y despliegue son manejados por Salesforce.
- Acceso al ecosistema npm para integraciones de terceros (Algolia, Coveo, Constructor, analytics, etc.).
- Separación de equipos de frontend y backend — ciclos de release independientes.

### Consideraciones

- Requiere expertise en React — el equipo necesita ingenieros frontend cómodos con desarrollo moderno en JavaScript/TypeScript.
- Los datos de comercio se acceden exclusivamente via SCAPI, que puede tener diferentes capacidades o características de latencia comparado con la Platform Script API interna usada por SFRA.
- Dos sistemas que entender: la aplicación frontend (PWA Kit) y el backend de comercio (B2C Commerce + Business Manager).
- Los límites de tamaño de bundle (400 MB total) requieren atención a la optimización de assets.

### Documentación Oficial

| Recurso | URL |
|---------|-----|
| Composable Storefront Overview | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/overview |
| PWA Kit Architecture | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/architecture.html |
| Getting Started | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/getting-started.html |
| Retail React App | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/retail-react-app.html |
| Deploying Bundles | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/pushing-and-deploying-bundles.html |
| SCAPI Reference | https://developer.salesforce.com/docs/commerce/commerce-api/references/ |
| commerce-sdk-react | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/commerce-sdk-react.html |
| PWA Kit GitHub Repository | https://github.com/SalesforceCommerceCloud/pwa-kit |

---

## 3. Headless Commerce

### Qué es Headless en el Contexto de B2C Commerce Cloud

Headless commerce es un patrón arquitectónico donde B2C Commerce Cloud opera exclusivamente como motor de comercio — exponiendo sus capacidades (catálogo, precios, promociones, carrito, checkout, órdenes, gestión de clientes) a través de APIs REST — mientras que la capa de presentación se construye, hostea y gestiona completamente fuera de la infraestructura de Salesforce.

En este modelo, no hay Managed Runtime, no hay PWA Kit, no hay frontend hosteado por Salesforce. La organización toma propiedad total de la capa de experiencia: eligiendo el framework de frontend, el proveedor de hosting, el CDN, el CMS, el motor de búsqueda y cada otro componente del stack. B2C Commerce Cloud se convierte en un servicio más dentro de una arquitectura compuesta.

Es aquí donde la plataforma se alinea con una estrategia de **arquitectura MACH**:

- **Microservices** — cada capacidad (comercio, contenido, búsqueda, pagos, personalización) es un servicio independiente con su propio ciclo de vida.
- **API-first** — toda la comunicación entre servicios ocurre a través de APIs bien definidas. B2C Commerce Cloud expone SCAPI y OCAPI como su superficie de integración.
- **Cloud-native** — los servicios corren en infraestructura cloud diseñada para elasticidad, resiliencia y distribución global.
- **Headless** — el frontend está completamente desacoplado de cualquier servicio backend, libre de consumir múltiples APIs y renderizar experiencias sin restricciones de plataforma.

Salesforce no participa en la MACH Alliance, pero la capa de APIs de B2C Commerce Cloud provee la base técnica para que una organización construya una arquitectura compatible con MACH usando Salesforce como motor de comercio.

### Modelo Arquitectónico

En una arquitectura headless, la organización ensambla el stack completo:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Navegador / App                              │
│  (Cualquier framework: Next.js, Nuxt, Astro, Remix, Angular,   │
│   Vue, Flutter, React Native, iOS/Android nativo, kiosko, IoT) │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                     BFF / API Gateway                             │
│  (Capa de orquestación custom — opcional pero común)             │
└──┬──────────────┬──────────────┬──────────────┬─────────────────┘
   │              │              │              │
   ▼              ▼              ▼              ▼
┌──────┐   ┌──────────┐   ┌─────────┐   ┌──────────────┐
│ CMS  │   │  Search  │   │Commerce │   │   Pagos      │
│      │   │          │   │ Engine  │   │              │
│Content│   │Algolia   │   │  B2C    │   │ Stripe      │
│ful   │   │Coveo     │   │Commerce │   │ Adyen       │
│Strapi│   │Construc- │   │ Cloud   │   │ Checkout    │
│Sanity│   │ tor      │   │ (SCAPI) │   │ .com        │
└──────┘   └──────────┘   └─────────┘   └──────────────┘
```

Cada servicio es independientemente:
- Desplegado y escalado
- Versionado y actualizado
- Monitoreado y mantenido
- Reemplazable si surge una mejor alternativa

### B2C Commerce Cloud como Motor de Comercio

En una arquitectura headless, B2C Commerce Cloud provee los servicios core de comercio a través de sus APIs:

**Shopper APIs (SCAPI)** — orientadas al consumidor, aseguradas via SLAS:
- Shopper Products (navegación de catálogo, detalle de producto, variaciones)
- Shopper Search (búsqueda de productos, navegación por categoría, refinamientos)
- Shopper Baskets (gestión de carrito, agregar/quitar items, cupones)
- Shopper Orders (colocación de órdenes, historial de órdenes)
- Shopper Customers (registro, login, perfiles, direcciones, instrumentos de pago)
- Shopper Promotions (promociones activas, ofertas impulsadas por campañas)
- Shopper Gift Certificates (validación y redención)
- Shopper Login (SLAS — gestión de tokens OAuth 2.0)

**OCAPI (Open Commerce API)** — legacy pero aún operativa para escenarios donde la cobertura de SCAPI es incompleta:
- Shop API (operaciones orientadas al shopper)
- Data API (operaciones de admin/back-office)

**SLAS (Shopper Login and API Access Service)** — capa de autenticación:
- Clientes públicos (navegador, apps móviles — sin secret)
- Clientes privados (BFF, del lado del servidor — con secret)
- Flujos de shoppers guest y registrados
- OAuth 2.0 con scopes configurables

La organización consume solo las APIs que necesita. Business Manager sigue siendo el back-office de merchandising — catálogos, precios, promociones y campañas se siguen gestionando ahí y se exponen a través de las APIs.

### El Rol del BFF (Backend for Frontend)

En la mayoría de las implementaciones headless, una capa BFF (Backend for Frontend) se ubica entre el frontend y los diversos servicios backend. Este no es un componente de Salesforce — es construido y mantenido por la organización.

El BFF cumple múltiples propósitos:

- **Orquestación de APIs** — combinar datos de múltiples servicios (comercio + CMS + búsqueda + personalización) en una única respuesta optimizada para el frontend.
- **Transformación de datos** — reformar las respuestas de API para que coincidan con la estructura de datos exacta que los componentes del frontend esperan.
- **Caching** — implementar estrategias de caché personalizadas por endpoint, reduciendo llamadas a APIs y mejorando tiempos de respuesta.
- **Gestión de autenticación** — manejar el ciclo de vida de tokens SLAS (adquisición, refresh, almacenamiento) del lado del servidor, manteniendo los secrets fuera del navegador.
- **Rate limiting y circuit breaking** — proteger contra fallas en cascada cuando un servicio downstream se degrada.
- **Abstracción de versionado de API** — aislar al frontend de breaking changes en versiones de APIs backend.

Implementaciones comunes de BFF:
- Node.js / Express o Fastify
- Next.js API routes
- Cloudflare Workers / Vercel Edge Functions
- AWS Lambda / API Gateway
- Capa de federación GraphQL (Apollo Gateway, Mesh)

### Libertad de Frontend

El modelo headless no impone restricciones sobre la capa de presentación. El frontend puede ser:

| Tecnología | Caso de uso |
|-----------|------------|
| Next.js (React) | Storefront web full-featured con SSR/SSG/ISR |
| Nuxt (Vue) | Storefront basado en Vue con server-side rendering |
| Astro | Storefronts con mucho contenido y mínimo JavaScript |
| Remix | Framework React enfocado en estándares web |
| Angular | Equipos enterprise con expertise en Angular |
| React Native / Flutter | Apps de comercio móvil nativas |
| Swift / Kotlin | Apps iOS / Android completamente nativas |
| SPA Custom | Cualquier framework JavaScript o JS vanilla |
| IoT / Kiosco | Dispositivos embebidos, punto de venta, displays inteligentes |

La elección depende completamente de las habilidades del equipo de la organización, los requisitos de performance, las necesidades de SEO y las plataformas objetivo.

### Gestión de Contenido en Headless

En este modelo, Page Designer (Business Manager) típicamente no se usa para gestión de contenido. En su lugar, la organización selecciona un CMS headless que sirve contenido via API:

- **Contentful** — contenido estructurado con una API rica y entrega por CDN
- **Sanity** — edición colaborativa en tiempo real con lenguaje de consulta GROQ
- **Strapi** — CMS headless open-source, self-hosted
- **Amplience** — gestión de contenido enfocada en comercio con scheduling
- **Contentstack** — CMS headless enterprise con automatización de workflows

El CMS gestiona contenido editorial (banners, landing pages, blog posts, contenido promocional), mientras que B2C Commerce gestiona contenido de comercio (productos, categorías, precios, promociones). El BFF o el frontend orquestan ambos en una experiencia unificada.

### Búsqueda y Descubrimiento

De forma similar, la búsqueda y descubrimiento de productos puede delegarse a servicios especializados:

- **Algolia** — búsqueda en tiempo real con tolerancia a typos, faceting y recomendaciones IA
- **Coveo** — búsqueda y relevancia impulsada por IA con ranking por machine learning
- **Constructor** — descubrimiento de productos con personalización y reglas de merchandising
- **Elasticsearch / OpenSearch** — infraestructura de búsqueda auto-gestionada

Estos servicios indexan datos de producto desde B2C Commerce (via API o data feeds) y proveen resultados de búsqueda directamente al frontend, bypasseando la búsqueda nativa de B2C Commerce por completo. Esto permite respuestas de búsqueda por debajo de 50ms y funcionalidades avanzadas como búsqueda visual, consultas en lenguaje natural y recomendaciones potenciadas por IA.

### Propiedad de la Infraestructura

En una arquitectura headless, la organización posee y opera:

| Aspecto | Responsabilidad |
|---------|----------------|
| Hosting del frontend | Vercel, Netlify, AWS CloudFront, Azure CDN, Cloudflare Pages, auto-gestionado |
| CDN / Edge caching | Configurado por proveedor — la invalidación de caché es tu responsabilidad |
| WAF / Seguridad | WAF del cloud provider, Cloudflare, Akamai, o reglas custom |
| Escalamiento | Políticas de auto-scaling, funciones serverless, orquestación de contenedores |
| CI/CD | GitHub Actions, GitLab CI, Jenkins, CircleCI — pipelines custom |
| Monitoreo | Datadog, New Relic, Grafana, stack de observabilidad custom |
| Tracking de errores | Sentry, Bugsnag, Rollbar |
| Performance | Monitoreo de Core Web Vitals, checks sintéticos, RUM |

Este es un cambio operativo significativo comparado con SFRA o Composable Storefront, donde Salesforce gestiona hosting, CDN, WAF y scaling.

### Flujo de Desarrollo

- **Desarrollo local completo** — todo el frontend corre localmente con control total sobre el dev server, el pipeline de build y las herramientas.
- **Tooling nativo del framework** — usar lo que el framework elegido provea (Next.js dev server, Vite, Turbopack, etc.).
- **API mocking** — los equipos pueden desarrollar contra APIs mock o instancias sandbox de B2C Commerce, desacoplando la velocidad del frontend de la disponibilidad del backend.
- **Despliegues independientes** — los releases del frontend son completamente independientes de los despliegues de B2C Commerce. Publicar cambios de frontend sin tocar el backend de comercio.
- **Estrategias multi-ambiente** — ambientes de desarrollo, staging, preview, producción gestionados a través de la infraestructura propia de la organización.
- **Testing en todos los niveles** — unitario, integración, e2e (Playwright, Cypress), regresión visual, performance (Lighthouse CI), accesibilidad (axe).

### Resumen del Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| Runtime | Cualquiera (Node.js, Deno, edge functions, serverless, contenedores) |
| Framework | Cualquiera (Next.js, Nuxt, Astro, Remix, Angular, custom) |
| Lenguaje | Cualquiera (TypeScript, JavaScript, Go, Rust para edge, etc.) |
| Librería UI | Cualquiera (Tailwind, Material UI, design system custom) |
| CMS | CMS Headless (Contentful, Sanity, Strapi, Amplience) |
| Búsqueda | Especializada (Algolia, Coveo, Constructor, OpenSearch) |
| API de Comercio | SCAPI + OCAPI (B2C Commerce Cloud) |
| Autenticación | SLAS (cliente público o privado) |
| BFF | Custom (Node.js, edge functions, GraphQL gateway) |
| Hosting | Cualquier cloud provider (Vercel, AWS, Azure, GCP, Cloudflare) |
| CDN | Gestionado por proveedor o custom (CloudFront, Fastly, Akamai) |
| CI/CD | Pipelines custom (GitHub Actions, GitLab CI, etc.) |
| Monitoreo | Stack de observabilidad custom |

### Ejemplos de Código

#### Next.js — Página de Detalle de Producto con ISR

```typescript
import { GetStaticProps, GetStaticPaths } from 'next'

interface Product {
    id: string
    name: string
    price: number
    description: string
    imageUrl: string
}

export const getStaticPaths: GetStaticPaths = async () => {
    const res = await fetch(
        `${process.env.SCAPI_BASE_URL}/product/shopper-products/v1/organizations/${process.env.ORG_ID}/products?siteId=${process.env.SITE_ID}&ids=top-seller-1,top-seller-2`,
        { headers: { Authorization: `Bearer ${await getAccessToken()}` } }
    )
    const data = await res.json()

    return {
        paths: data.data.map((p: Product) => ({ params: { pid: p.id } })),
        fallback: 'blocking'
    }
}

export const getStaticProps: GetStaticProps = async ({ params }) => {
    const token = await getAccessToken()
    const res = await fetch(
        `${process.env.SCAPI_BASE_URL}/product/shopper-products/v1/organizations/${process.env.ORG_ID}/products/${params?.pid}?siteId=${process.env.SITE_ID}`,
        { headers: { Authorization: `Bearer ${token}` } }
    )
    const product = await res.json()

    return { props: { product }, revalidate: 60 }
}

export default function ProductPage({ product }: { product: Product }) {
    return (
        <div className="max-w-6xl mx-auto p-6 grid grid-cols-1 md:grid-cols-2 gap-8">
            <img src={product.imageUrl} alt={product.name} className="w-full rounded-lg" />
            <div>
                <h1 className="text-3xl font-bold">{product.name}</h1>
                <p className="text-2xl mt-4">${product.price}</p>
                <p className="mt-4 text-gray-600">{product.description}</p>
                <button className="mt-6 bg-blue-600 text-white px-8 py-3 rounded-lg">
                    Agregar al Carrito
                </button>
            </div>
        </div>
    )
}
```

#### BFF — Gestión de Tokens SLAS (Node.js)

```typescript
import { Router } from 'express'

const router = Router()

const SLAS_BASE = process.env.SLAS_BASE_URL
const CLIENT_ID = process.env.SLAS_CLIENT_ID
const CLIENT_SECRET = process.env.SLAS_CLIENT_SECRET
const ORG_ID = process.env.COMMERCE_ORG_ID
const SITE_ID = process.env.COMMERCE_SITE_ID

router.post('/auth/guest-token', async (req, res) => {
    const response = await fetch(
        `${SLAS_BASE}/shopper/auth/v1/organizations/${ORG_ID}/oauth2/token`,
        {
            method: 'POST',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
            body: new URLSearchParams({
                grant_type: 'client_credentials',
                client_id: CLIENT_ID,
                client_secret: CLIENT_SECRET,
                channel_id: SITE_ID
            })
        }
    )

    const token = await response.json()
    res.json({ access_token: token.access_token, expires_in: token.expires_in })
})

router.post('/auth/login', async (req, res) => {
    const { username, password } = req.body

    const response = await fetch(
        `${SLAS_BASE}/shopper/auth/v1/organizations/${ORG_ID}/oauth2/token`,
        {
            method: 'POST',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
            body: new URLSearchParams({
                grant_type: 'urn:demandware:params:oauth:grant-type:credentials',
                client_id: CLIENT_ID,
                client_secret: CLIENT_SECRET,
                channel_id: SITE_ID,
                username,
                password
            })
        }
    )

    const token = await response.json()
    res.json({ access_token: token.access_token, refresh_token: token.refresh_token })
})

export default router
```

#### Orquestación de APIs — Combinando Commerce + CMS

```typescript
import { getCommerceProduct } from './services/commerce'
import { getCmsContent } from './services/cms'
import { getSearchRecommendations } from './services/search'

async function getProductPageData(productId: string) {
    const [product, editorial, recommendations] = await Promise.all([
        getCommerceProduct(productId),
        getCmsContent(`product-${productId}-editorial`),
        getSearchRecommendations(productId)
    ])

    return {
        product: {
            id: product.id,
            name: product.name,
            price: product.price,
            images: product.imageGroups[0].images,
            variants: product.variants,
            inventory: product.inventory
        },
        editorial: {
            headline: editorial.fields.headline,
            body: editorial.fields.richText,
            media: editorial.fields.heroImage.url
        },
        recommendations: recommendations.hits.map(hit => ({
            id: hit.productId,
            name: hit.productName,
            price: hit.price,
            image: hit.thumbUrl
        }))
    }
}
```

### Cuándo Usar Headless

Headless es el tercer grado de implementación en B2C Commerce Cloud. Es el camino para organizaciones que requieren libertad arquitectónica completa y están preparadas para ser dueñas del stack tecnológico completo más allá del motor de comercio. Habilita una estrategia de arquitectura MACH donde B2C Commerce Cloud sirve como el microservicio de comercio dentro de un ecosistema compuesto más amplio.

### Fortalezas

- Libertad tecnológica completa — sin restricciones en decisiones de framework, lenguaje, hosting o infraestructura.
- Verdadera arquitectura MACH — cada componente es independientemente desplegable, escalable y reemplazable.
- Multi-canal por diseño — las mismas APIs de comercio sirven web, apps móviles, IoT, kioscos, asistentes de voz y cualquier touchpoint futuro.
- Composición best-of-breed — seleccionar el servicio óptimo para cada capacidad (búsqueda, CMS, personalización, pagos) en lugar de aceptar la solución de un solo vendor para todo.
- Escalamiento independiente — frontend, BFF y motor de comercio escalan independientemente según sus propios patrones de tráfico.
- Máximo control de performance — edge rendering, generación estática, regeneración incremental y estrategias de caché custom están todas disponibles.
- Flexibilidad de vendor — el frontend no está bloqueado en la infraestructura de Salesforce. Si el motor de comercio necesita cambiar, solo la capa de integración API se ve afectada.

### Consideraciones

- Responsabilidad total de infraestructura — la organización debe provisionar, asegurar, monitorear y mantener todos los componentes no relacionados con comercio.
- Mayor complejidad operativa — más partes móviles significa más puntos potenciales de falla, más monitoreo, más carga de on-call.
- Requiere equipo de ingeniería fuerte — este modelo demanda expertise en infraestructura cloud, diseño de APIs, arquitectura de frontend, DevOps y sistemas distribuidos.
- Mayor time-to-market para el lanzamiento inicial — ensamblar el stack desde cero requiere más inversión inicial que empezar con SFRA o Composable Storefront.
- Sin Page Designer — la gestión de contenido requiere un CMS headless separado y workflows editoriales custom.
- Rate limits y cuotas de API — las APIs de B2C Commerce Cloud tienen rate limits que deben considerarse en las estrategias de caching y orquestación.
- Mantenimiento de integraciones — cuando B2C Commerce Cloud lanza actualizaciones o deprecaciones de API, la organización es responsable de adaptarse.

### Documentación Oficial

| Recurso | URL |
|---------|-----|
| B2C Commerce API (SCAPI) | https://developer.salesforce.com/docs/commerce/commerce-api/references/ |
| SLAS Authorization | https://developer.salesforce.com/docs/commerce/commerce-api/guide/authorization-for-shopper-apis.html |
| OCAPI Reference | https://developer.salesforce.com/docs/commerce/b2c-commerce/references/b2c-commerce-ocapi/ |
| B2C Commerce Developer Resources | https://salesforcecommercecloud.github.io/b2c-dev-doc/ |
| commerce-sdk (TypeScript) | https://github.com/SalesforceCommerceCloud/commerce-sdk |
| commerce-sdk-isomorphic | https://github.com/SalesforceCommerceCloud/commerce-sdk-isomorphic |
