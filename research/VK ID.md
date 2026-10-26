# Вход через VK ID

Заготовка, которую выдал кабинет VK ID для бизнеса (id.vk.com/business/go) при создании приложения.

## Приложение

- ID приложения: `54857083` (публичный, не секрет). Защищённый ключ приложения в документы не записывать — только переменные окружения.
- Платформа «Веб»:
  - базовый домен — `у-истока.рф` (punycode `xn----8sbxitrhe.xn--p1ai`);
  - доверенный Redirect URL — `https://auth.у-истока.рф/login/vk/callback` (punycode `https://auth.xn----8sbxitrhe.xn--p1ai/login/vk/callback`). Путь черновой, утвердим в спеке входа.
- Платформа Android: название пакета `ru.uistoka.app`, SHA-1 — от временного ключа, которым ничего не подписывается. Когда появится приложение, заменить на отпечаток настоящего ключа подписи.

## Код виджета из кабинета

```html
<div>
  <script nonce="csp_nonce" src="https://unpkg.com/@vkid/sdk@<3.0.0/dist-sdk/umd/index.js"></script>
  <script nonce="csp_nonce" type="text/javascript">
    if ('VKIDSDK' in window) {
      const VKID = window.VKIDSDK;

      VKID.Config.init({
        app: 54857083,
        redirectUrl: 'https://auth.у-истока.рф/login/vk/callback',
        responseMode: VKID.ConfigResponseMode.Callback,
        source: VKID.ConfigSource.LOWCODE,
        scope: '', // Заполните нужными доступами по необходимости
      });

      const oneTap = new VKID.OneTap();

      oneTap.render({
        container: document.currentScript.parentElement,
        showAlternativeLogin: true
      })
      .on(VKID.WidgetEvents.ERROR, vkidOnError)
      .on(VKID.OneTapInternalEvents.LOGIN_SUCCESS, function (payload) {
        const code = payload.code;
        const deviceId = payload.device_id;

        VKID.Auth.exchangeCode(code, deviceId)
          .then(vkidOnSuccess)
          .catch(vkidOnError);
      });

      function vkidOnSuccess(data) {
        // Обработка полученного результата
      }

      function vkidOnError(error) {
        // Обработка ошибки
      }
    }
  </script>
</div>
```

## Заметки к реализации

- В заготовке `VKID.Auth.exchangeCode` меняет код на токен прямо в браузере. Для нас нужен обмен на сервере (сервис входа): браузер передаёт `code` и `device_id`, сервер обменивает их через `id.vk.ru/oauth2/auth` с PKCE (`code_verifier`) и берёт профиль из `id.vk.ru/oauth2/user_info`. Токены VK в браузере не остаются.
- Версию SDK закрепить точной (`@vkid/sdk@2.x.y`), а не диапазоном `<3.0.0`, или поставить пакет через npm.
- `scope` пустой — достаточно имени и аватара; почту и телефон не запрашиваем, пока не решено, что они нужны.
