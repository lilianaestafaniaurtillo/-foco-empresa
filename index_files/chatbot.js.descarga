/**
 * Loader vanilla-JS del widget SOFIA. Se embebe en cualquier página del
 * Banco con:
 *
 *   <script src="https://<host-de-este-frontend>/sofia-widget.js"
 *           data-base-url="https://<host-de-este-frontend>"
 *           data-channel="web"></script>
 *
 * Crea un lanzador flotante (esfera animada) con una tarjeta de saludo en
 * forma de píldora a su lado, y un iframe apuntando a
 * `${data-base-url}/login?channel=...`. Escucha los mensajes
 * MINIMIZE_SOFIA_WIDGET / CLOSE_SOFIA_WIDGET / SOFIA_APP_READY /
 * SOFIA_SERVICE_MAINTENANCE que ya emite la app (ver ChatContent.tsx /
 * SofiaLogin.tsx / maintenance/page.tsx) para mostrar/ocultar el iframe. No
 * usa `postMessage` hacia afuera: solo escucha, nunca publica datos del host.
 *
 * Versiones anteriores: scripts/backup-sofia-widget.js (lanzador circular +
 * burbuja de saludo, de donde sale la lógica de la tarjeta) y
 * scripts/backup-2-sofia-widget.js (barra "Chatea con" + lágrima).
 */
(function () {
  // Mantener en sync con `agentName` en src/consts/branding.ts: este script
  // es estático y no pasa por el bundler de Next.js, así que no puede
  // importar ese archivo.
  var WIDGET_AGENT_NAME = 'Dani';
  // Esfera de insumos/Dani.gif como WebP animado, con la luminancia llevada a
  // la paleta morada. El GIF tiene un corte visible al reiniciar; en el WebP
  // los primeros 45 cuadros se funden con los últimos 45 (que se descartan),
  // así que el bucle es continuo: 136 cuadros de 30 ms.
  var WIDGET_SPHERE_PATH = '/assets/img/esfera-dani.webp';
  var GREETING_DELAY_MS = 300;
  var CARD_TRANSITION_MS = 800;

  // Paleta de src/styles/global.css. Mantener en sync a mano, por el mismo
  // motivo que las constantes de arriba: la página anfitriona no tiene esas
  // variables CSS.
  var COLOR_PURPLE = '#3F29AB'; // --principal-purple
  var COLOR_PURPLE_LIGHT = '#8C7FCD'; // --principal-purple-light
  var COLOR_TEXT = '#1d1d1f';
  var SPHERE_OUTLINE = 'rgba(6,5,11,.85)';
  var COLOR_MAINTENANCE_TEXT = '#616161'; // --black-gray
  // Borde de la tarjeta: degradado lineal que gira en sentido horario, una
  // vuelta por ciclo del GIF (181 cuadros x 30 ms). Las paradas salen de
  // medir el borde del GIF en el cuadro 0: morado a la izquierda, lavanda al
  // centro, casi blanco a la derecha.
  var RING_TURN_MS = 5430;
  var RING_GRADIENT =
    'linear-gradient(90deg,' + COLOR_PURPLE + ' 0%,' + COLOR_PURPLE_LIGHT + ' 35%,#f9f9fc 85%,#f9f9fc 100%)';
  var RING_GRADIENT_MAINTENANCE = 'linear-gradient(90deg,#9e9e9e 0%,#d9d9d9 35%,#f5f5f5 85%,#f5f5f5 100%)';

  // Red de seguridad: normalmente la apertura la dispara SOFIA_APP_READY (o
  // el paso a gris lo dispara SOFIA_SERVICE_MAINTENANCE). Este timer solo
  // actúa si ninguna de las dos señales llega tras un clic (ej. la
  // navegación del iframe falla por red) para no dejar el lanzador
  // deshabilitado para siempre.
  //
  // 35000 y no menos: con un valor más bajo este respaldo compite con
  // checkGatewayHealth() (gatewayClient.ts) y puede ganarle la carrera: 3
  // intentos x 10s (HEALTH_TIMEOUT_MS) + 1s + 2s de backoff entre ellos =
  // 33000ms de peor caso ante un gateway lento-pero-no-caído, y el respaldo
  // abriría el iframe en blanco en vez de mostrarlo en gris. Si
  // HEALTH_TIMEOUT_MS/HEALTH_RETRY_ATTEMPTS/HEALTH_BACKOFF_MS cambian en
  // gatewayClient.ts, este valor debe recalcularse (peor caso =
  // intentos*timeout + suma de backoffs) y subirse por encima, nunca bajarse
  // ni desactivarse.
  var OPEN_FALLBACK_TIMEOUT_MS = 35000;

  // Medidas: tarjeta de 76px de alto y esfera de 58px (el 76% del alto, como
  // en el GIF), con el mismo margen arriba y a la derecha. La esfera queda
  // siempre en el mismo lugar, se vea o no la tarjeta. La tarjeta mide lo
  // mismo con el saludo y con el aviso de mantenimiento: ancho y alto fijos
  // (el ancho es el que ocupaba el saludo), y el aviso se recorta a dos
  // líneas en vez de estirarla.
  var EDGE_OFFSET = 24;
  var CARD_WIDTH = 304;
  var CARD_HEIGHT = 76;
  var SPHERE_SIZE = 58;
  var SPHERE_INSET = (CARD_HEIGHT - SPHERE_SIZE) / 2;
  var RING_WIDTH = 3;

  var currentScript = document.currentScript;
  if (!currentScript) return;

  var baseUrl = (currentScript.getAttribute('data-base-url') || '').replace(/\/+$/, '');
  var channel = currentScript.getAttribute('data-channel') || 'web';

  if (!baseUrl) {
    console.error('[sofia-widget] falta data-base-url en el <script> de embebido');
    return;
  }

  var FONT_FAMILY = 'Poppins,Arial,Helvetica,sans-serif';
  var MAINTENANCE_FALLBACK_TEXT = 'En mantenimiento';

  // Las animaciones (giro del borde y spinner) necesitan @keyframes, que no
  // se pueden meter en cssText. Un <style> sin nonce no se aplica bajo una
  // CSP con style-src restrictivo (sin 'unsafe-inline') -- si la página
  // anfitriona tiene una así, el borde queda quieto y el spinner no gira,
  // pero todo lo demás funciona. Si Next.js (u otro origen) le puso un nonce
  // a esta misma etiqueta <script>, se reusa acá.
  var widgetStyle = document.createElement('style');
  if (currentScript.nonce) {
    widgetStyle.nonce = currentScript.nonce;
  }
  widgetStyle.textContent =
    '@keyframes sofiaWidgetSpin{to{transform:rotate(360deg);}}' +
    '.sofia-widget-ring{animation:sofiaWidgetSpin ' + RING_TURN_MS + 'ms linear infinite;}' +
    '@media (prefers-reduced-motion:reduce){.sofia-widget-ring{animation:none;}}';
  document.head.appendChild(widgetStyle);

  // Esfera: es el lanzador, siempre visible (salvo con el chat abierto).
  var launcher = document.createElement('button');
  launcher.type = 'button';
  launcher.setAttribute('aria-label', 'Abrir chat ' + WIDGET_AGENT_NAME.toUpperCase());
  launcher.style.cssText =
    'position:fixed;bottom:' + (EDGE_OFFSET + SPHERE_INSET) + 'px;right:' + (EDGE_OFFSET + SPHERE_INSET) + 'px;' +
    'z-index:2147483001;display:flex;align-items:center;justify-content:center;' +
    'width:' + SPHERE_SIZE + 'px;height:' + SPHERE_SIZE + 'px;padding:0;margin:0;border-radius:50%;' +
    'border:1px solid ' + SPHERE_OUTLINE + ';box-sizing:border-box;cursor:pointer;' +
    'background:' + COLOR_PURPLE + ';box-shadow:2px 4px 8px rgba(0,0,0,.35);' +
    'transition:background .35s ease;-webkit-tap-highlight-color:transparent;';

  var sphereImage = document.createElement('img');
  sphereImage.src = baseUrl + WIDGET_SPHERE_PATH;
  sphereImage.alt = '';
  sphereImage.style.cssText =
    'position:absolute;inset:0;width:100%;height:100%;object-fit:cover;border-radius:50%;pointer-events:none;' +
    'transition:filter .35s ease;';

  // Loader: mientras se verifica mantenimiento tras un clic, el propio borde
  // de la esfera pasa a ser el indicador de carga -- un anillo de 3px sobre
  // el contorno (inset -1px lo cubre exacto) con un tramo morado claro
  // girando. La esfera sigue animándose adentro, sin nada encima.
  var launcherLoader = document.createElement('span');
  launcherLoader.style.cssText =
    'position:absolute;inset:-1px;display:none;border-radius:50%;box-sizing:border-box;' +
    'border:3px solid ' + SPHERE_OUTLINE + ';border-top-color:' + COLOR_PURPLE_LIGHT + ';' +
    'animation:sofiaWidgetSpin .8s linear infinite;pointer-events:none;';

  launcher.appendChild(sphereImage);
  launcher.appendChild(launcherLoader);

  function setLauncherLoading(isLoading) {
    launcherLoader.style.display = isLoading ? 'block' : 'none';
  }

  // Tarjeta de saludo: píldora de vidrio blanco con el borde morado girando. La esfera
  // se dibuja encima de su extremo derecho (por eso el padding derecho
  // reserva su lugar). Es permanente: aparece GREETING_DELAY_MS después de
  // cargar la página anfitriona y queda visible mientras se vea la esfera;
  // solo se oculta con el chat abierto. Clic en el cuerpo abre el chat.
  //
  // Entrada de derecha a izquierda: un recorte (`clip-path`) que se abre
  // desde el borde derecho, como si la tarjeta saliera de detrás de la
  // esfera, más un desplazamiento corto y un fundido. Los insets negativos
  // dejan fuera del recorte la sombra de la tarjeta.
  var CARD_SLIDE_PX = 24;
  var GLASS_BACKGROUND = 'rgba(255,255,255,.8)';
  var GLASS_BACKDROP = 'blur(16px) saturate(180%)';
  var CARD_EASING = 'cubic-bezier(.22,1,.36,1)';
  var CARD_TRANSITION =
    'opacity ' + CARD_TRANSITION_MS + 'ms ease,transform ' + CARD_TRANSITION_MS + 'ms ' + CARD_EASING + ',' +
    'clip-path ' + CARD_TRANSITION_MS + 'ms ' + CARD_EASING;
  var CARD_FADE_OUT_MS = 300;
  var CARD_CLIP_HIDDEN = 'inset(-30px -30px -30px 100%)';
  var CARD_CLIP_SHOWN = 'inset(-30px -30px -30px -30px)';
  var greeting = document.createElement('div');
  greeting.setAttribute('role', 'button');
  greeting.setAttribute('tabindex', '0');
  greeting.setAttribute('aria-label', 'Soy ' + WIDGET_AGENT_NAME + '. ¿En qué puedo ayudarte?');
  greeting.style.cssText =
    'position:fixed;bottom:' + EDGE_OFFSET + 'px;right:' + EDGE_OFFSET + 'px;z-index:2147483000;' +
    'display:none;box-sizing:border-box;height:' + CARD_HEIGHT + 'px;' +
    'width:min(' + CARD_WIDTH + 'px, calc(100vw - ' + EDGE_OFFSET * 2 + 'px));padding:' + RING_WIDTH + 'px;border-radius:' + CARD_HEIGHT + 'px;' +
    'overflow:hidden;isolation:isolate;cursor:pointer;box-shadow:0 6px 18px rgba(0,0,0,.18);' +
    // Vidrio estilo iOS: el desenfoque va en la tarjeta y no en el cuerpo
    // blanco, porque el clip-path/opacity de la tarjeta la vuelven la raíz
    // del backdrop de sus hijos: en el cuerpo solo vería la propia tarjeta,
    // no la página de atrás.
    '-webkit-backdrop-filter:' + GLASS_BACKDROP + ';backdrop-filter:' + GLASS_BACKDROP + ';' +
    'opacity:0;transform:translateX(' + CARD_SLIDE_PX + 'px);clip-path:' + CARD_CLIP_HIDDEN + ';' +
    'transition:' + CARD_TRANSITION + ';';

  // Capa del borde: ocupa toda la tarjeta pero una máscara deja ver solo el
  // anillo de RING_WIDTH px (todo menos el content-box). Hace falta porque
  // el cuerpo es translúcido: sin la máscara, el degradado girando se vería
  // a través del vidrio.
  var ringLayer = document.createElement('span');
  ringLayer.style.cssText =
    'position:absolute;inset:0;z-index:-1;box-sizing:border-box;padding:' + RING_WIDTH + 'px;' +
    'border-radius:' + CARD_HEIGHT + 'px;overflow:hidden;pointer-events:none;' +
    '-webkit-mask:linear-gradient(#000 0 0) content-box,linear-gradient(#000 0 0);' +
    '-webkit-mask-composite:xor;mask:linear-gradient(#000 0 0) content-box exclude,linear-gradient(#000 0 0);';

  // Cuadrado del tamaño de la diagonal de la tarjeta, centrado detrás de
  // ella y girando: así el degradado lineal recorre el borde completo como
  // en el GIF (de punta a punta en horizontal, solo el tramo central en
  // vertical). El tamaño se recalcula cada vez que la tarjeta aparece.
  var ring = document.createElement('span');
  ring.className = 'sofia-widget-ring';
  ring.style.cssText =
    'position:absolute;top:50%;left:50%;width:0;height:0;pointer-events:none;' +
    'background:' + RING_GRADIENT + ';';

  var greetingBody = document.createElement('div');
  greetingBody.style.cssText =
    'display:flex;flex-direction:column;justify-content:center;box-sizing:border-box;' +
    'height:' + (CARD_HEIGHT - RING_WIDTH * 2) + 'px;border-radius:' + CARD_HEIGHT + 'px;' +
    'padding:8px ' + (SPHERE_SIZE + SPHERE_INSET + 12 - RING_WIDTH) + 'px 8px 22px;' +
    'background:' + GLASS_BACKGROUND + ';font-family:' + FONT_FAMILY + ';';

  var greetingTitle = document.createElement('div');
  greetingTitle.textContent = 'Soy ' + WIDGET_AGENT_NAME;
  greetingTitle.style.cssText =
    'color:' + COLOR_PURPLE + ';font-weight:600;font-size:22px;line-height:1.2;text-align:center;white-space:nowrap;' +
    'overflow:hidden;text-overflow:ellipsis;';

  var greetingSubtitle = document.createElement('div');
  greetingSubtitle.textContent = '¿En qué puedo ayudarte?';
  greetingSubtitle.style.cssText =
    'color:' + COLOR_TEXT + ';font-weight:400;font-size:15px;line-height:1.3;white-space:nowrap;' +
    'overflow:hidden;text-overflow:ellipsis;';

  greetingBody.appendChild(greetingTitle);
  greetingBody.appendChild(greetingSubtitle);
  ringLayer.appendChild(ring);
  greeting.appendChild(ringLayer);
  greeting.appendChild(greetingBody);

  // Sin `src` hasta el primer clic: así la verificación de mantenimiento
  // ocurre cuando el usuario intenta abrir el chat, no en segundo plano
  // apenas carga la página anfitriona.
  var frame = document.createElement('iframe');
  frame.title = WIDGET_AGENT_NAME.toUpperCase();
  frame.style.cssText =
    'position:fixed;bottom:24px;right:24px;width:400px;height:700px;max-width:calc(100vw - 32px);' +
    'max-height:calc(100vh - 32px);border:none;border-radius:16px;box-shadow:0 8px 24px rgba(0,0,0,.25);' +
    'z-index:2147483002;display:none;';

  var greetingTimer = null;
  var greetingHideTimer = null;
  var isMaintenance = false;
  var pendingCheckTimer = null;
  var isMounted = false;
  // true entre un MINIMIZE_SOFIA_WIDGET y el siguiente clic en el lanzador:
  // el iframe sigue vivo (misma sesión, mismo estado del chat) solo oculto
  // por CSS, así que reabrir no debe tocar `frame.src` ni repetir el
  // chequeo de mantenimiento -- eso recargaría el chat desde cero.
  var isMinimized = false;
  // true mientras el iframe está a la vista: la tarjeta no debe aparecer.
  // Sin este guard, si la entrada ya había arrancado justo cuando llega
  // SOFIA_APP_READY, la tarjeta quedaría pisando el iframe.
  var isWidgetOpen = false;

  // Estado visible/oculto de la tarjeta. `isInstant`: sin transición -- se
  // apaga, se aplica el estado, se fuerza un reflow para que el navegador lo
  // tome ya, y se devuelve la transición completa para la próxima entrada.
  function setCardState(isShown, isInstant) {
    if (isInstant) greeting.style.transition = 'none';
    greeting.style.opacity = isShown ? '1' : '0';
    greeting.style.transform = isShown ? 'translateX(0)' : 'translateX(' + CARD_SLIDE_PX + 'px)';
    greeting.style.clipPath = isShown ? CARD_CLIP_SHOWN : CARD_CLIP_HIDDEN;
    if (isInstant) {
      void greeting.offsetWidth;
      greeting.style.transition = CARD_TRANSITION;
    }
  }

  function sizeRing() {
    var width = greeting.offsetWidth;
    var height = greeting.offsetHeight;
    var side = Math.ceil(Math.sqrt(width * width + height * height));
    ring.style.width = side + 'px';
    ring.style.height = side + 'px';
    ring.style.margin = (-side / 2) + 'px 0 0 ' + (-side / 2) + 'px';
  }

  // `isInstant`: al cerrar o minimizar el chat la tarjeta ya está en su
  // lugar, sin volver a entrar; la entrada animada es solo para la primera
  // aparición tras cargar la página (y el paso a mantenimiento).
  function showGreeting(isInstant) {
    if (isWidgetOpen) return;
    if (greetingTimer) {
      clearTimeout(greetingTimer);
      greetingTimer = null;
    }
    if (greetingHideTimer) {
      clearTimeout(greetingHideTimer);
      greetingHideTimer = null;
    }
    greeting.style.display = 'block';
    sizeRing();
    if (isInstant === true) {
      setCardState(true, true);
      return;
    }
    // Doble rAF: deja que el navegador pinte el estado inicial (recortado y
    // desplazado) antes de pasar al final, si no la transición no se dispara.
    requestAnimationFrame(function () {
      requestAnimationFrame(function () {
        setCardState(true, false);
      });
    });
  }

  // `isInstant`: al abrir el chat la tarjeta queda tapada por el iframe, así
  // que se vuelve al estado inicial sin animación, lista para la próxima
  // entrada. Sin él (paso a mantenimiento) se recoge con la misma transición.
  function hideGreeting(isInstant) {
    if (greetingTimer) {
      clearTimeout(greetingTimer);
      greetingTimer = null;
    }
    if (greetingHideTimer) clearTimeout(greetingHideTimer);
    setCardState(false, isInstant === true);
    greetingHideTimer = setTimeout(function () {
      greetingHideTimer = null;
      greeting.style.display = 'none';
    }, isInstant === true ? 0 : CARD_TRANSITION_MS);
  }

  function scheduleGreeting() {
    if (isMaintenance) return;
    if (greetingTimer) clearTimeout(greetingTimer);
    greetingTimer = setTimeout(showGreeting, GREETING_DELAY_MS);
  }

  function openWidget() {
    frame.style.display = 'block';
    launcher.style.display = 'none';
    isMinimized = false;
    isWidgetOpen = true;
    hideGreeting(true);
  }

  // Punto de entrada real de los clics: en lugar de abrir el iframe de una,
  // dispara una navegación fresca a /login (donde vive la verificación de
  // mantenimiento) y espera la señal real de la app: SOFIA_APP_READY (login
  // montó, está sano → abre) o SOFIA_SERVICE_MAINTENANCE (pasa a gris). El
  // iframe permanece oculto hasta que se sepa cuál de las dos ocurrió.
  //
  // Excepción: si el widget estaba minimizado, el iframe ya tiene una
  // sesión viva -- se reabre tal cual, sin tocar `frame.src` ni repetir el
  // chequeo de mantenimiento.
  function checkAndOpen() {
    if (isMaintenance || pendingCheckTimer) return;

    if (isMinimized) {
      openWidget();
      return;
    }

    launcher.disabled = true;
    setLauncherLoading(true);
    frame.src = baseUrl + '/login?channel=' + encodeURIComponent(channel);

    pendingCheckTimer = setTimeout(function () {
      pendingCheckTimer = null;
      launcher.disabled = false;
      setLauncherLoading(false);
      openWidget();
    }, OPEN_FALLBACK_TIMEOUT_MS);
  }

  function confirmHealthyAndOpen() {
    // Si no había un chequeo en curso, es un mensaje fuera de secuencia
    // (ej. el usuario ya cerró el widget); se ignora.
    if (!pendingCheckTimer) return;
    clearTimeout(pendingCheckTimer);
    pendingCheckTimer = null;
    launcher.disabled = false;
    setLauncherLoading(false);
    openWidget();
  }

  function closeWidget() {
    frame.style.display = 'none';
    // 'flex', no 'block': el lanzador declara `display:flex` en su cssText
    // inicial para centrar el spinner sobre la esfera.
    launcher.style.display = 'flex';
    isMinimized = false;
    isWidgetOpen = false;
    frame.src = baseUrl + '/login?channel=' + encodeURIComponent(channel);

    // La tarjeta ya está, sin volver a entrar.
    if (!isMaintenance) showGreeting(true);
  }

  function minimizeWidget() {
    frame.style.display = 'none';
    launcher.style.display = 'flex';
    isMinimized = true;
    isWidgetOpen = false;
    if (!isMaintenance) showGreeting(true);
  }

  // Cuando el gateway no responde, src/proxy.js sirve /maintenance dentro
  // del iframe y esa página emite este mensaje (maintenance/page.tsx) con el
  // motivo en `message`. En lugar de dejar el iframe en una pantalla en
  // blanco, se cierra el widget, la esfera queda deshabilitada, todo lo
  // morado (esfera, borde y título) pasa a gris y la tarjeta reaparece con
  // el motivo en lugar del saludo. Solo `textContent`, nunca HTML.
  function showMaintenance(message) {
    isMaintenance = true;
    if (pendingCheckTimer) {
      clearTimeout(pendingCheckTimer);
      pendingCheckTimer = null;
    }

    frame.style.display = 'none';
    isWidgetOpen = false;
    isMinimized = false;
    launcher.style.display = 'flex';
    launcher.disabled = true;
    launcher.style.cursor = 'not-allowed';
    launcher.style.background = '#9e9e9e';
    launcher.setAttribute('aria-label', 'Servicio en mantenimiento');
    setLauncherLoading(false);
    sphereImage.style.filter = 'grayscale(1)';

    var maintenanceText = typeof message === 'string' && message ? message : MAINTENANCE_FALLBACK_TEXT;

    var applyMaintenanceContent = function () {
      greeting.style.cursor = 'default';
      greeting.removeAttribute('role');
      greeting.removeAttribute('tabindex');
      greeting.setAttribute('aria-label', maintenanceText);
      ring.style.background = RING_GRADIENT_MAINTENANCE;
      // El motivo puede ser largo: envuelve dentro del mismo ancho del
      // saludo y se corta a dos líneas con "…", así la tarjeta no cambia de
      // tamaño. El texto completo queda en `aria-label` y en `title`.
      greeting.title = maintenanceText;
      greetingTitle.textContent = maintenanceText;
      greetingTitle.style.color = COLOR_MAINTENANCE_TEXT;
      greetingTitle.style.fontSize = '15px';
      greetingTitle.style.fontWeight = '500';
      greetingTitle.style.whiteSpace = 'normal';
      greetingTitle.style.display = '-webkit-box';
      greetingTitle.style.webkitBoxOrient = 'vertical';
      greetingTitle.style.webkitLineClamp = '2';
      greetingSubtitle.style.display = 'none';
      // Misma entrada que el saludo inicial: misma espera y mismo recorte
      // de derecha a izquierda.
      greetingTimer = setTimeout(showGreeting, GREETING_DELAY_MS);
    };

    // Si el saludo ya estaba en pantalla, se apaga con un fundido corto (solo
    // opacidad: recogerlo con el recorte se veía brusco), se vuelve al estado
    // inicial sin animación y el aviso entra igual que el saludo.
    if (greeting.style.display === 'block') {
      if (greetingTimer) {
        clearTimeout(greetingTimer);
        greetingTimer = null;
      }
      greeting.style.transition = 'opacity ' + CARD_FADE_OUT_MS + 'ms ease';
      greeting.style.opacity = '0';
      setTimeout(function () {
        hideGreeting(true);
        applyMaintenanceContent();
      }, CARD_FADE_OUT_MS);
    } else {
      if (greetingTimer) {
        clearTimeout(greetingTimer);
        greetingTimer = null;
      }
      applyMaintenanceContent();
    }
  }

  launcher.addEventListener('click', checkAndOpen);
  greeting.addEventListener('click', checkAndOpen);
  greeting.addEventListener('keydown', function (event) {
    if (event.key === 'Enter' || event.key === ' ') {
      event.preventDefault();
      checkAndOpen();
    }
  });
  window.addEventListener('resize', function () {
    if (greeting.style.display === 'block') {
      sizeRing();
    }
  });

  window.addEventListener('message', function (event) {
    if (event.source !== frame.contentWindow) return;
    var action = event.data && event.data.action;
    if (action === 'MINIMIZE_SOFIA_WIDGET') minimizeWidget();
    if (action === 'CLOSE_SOFIA_WIDGET') closeWidget();
    if (action === 'SOFIA_APP_READY') confirmHealthyAndOpen();
    if (action === 'SOFIA_SERVICE_MAINTENANCE') showMaintenance(event.data && event.data.message);
  });

  // Puede llamarse dos veces (DOMContentLoaded + readyState 'interactive'
  // cuando el script corre antes de que termine el parseo); solo monta una.
  function mountWidget() {
    if (isMounted) return;
    isMounted = true;
    document.body.appendChild(frame);
    document.body.appendChild(greeting);
    document.body.appendChild(launcher);
    scheduleGreeting();
  }

  document.addEventListener('DOMContentLoaded', mountWidget);

  if (document.readyState === 'complete' || document.readyState === 'interactive') {
    mountWidget();
  }
})();
