# Nastavení vlastního Attestation Serveru pro GrapheneOS (Auditor)

Tento návod popisuje, jak úspěšně spárovat vlastní Attestation Server s telefony GrapheneOS (např. Pixel 10) pomocí aplikace Auditor.

## Hlavní překážky (Proč to nefunguje "out of the box")
Systém Auditor je navržen tak, aby byl maximálně bezpečný, což s sebou přináší dvě striktní restrikce:
1. **Oficiální aplikace** pevně věří pouze doméně `attestation.app` a Let's Encrypt certifikátům. Odmítne naskenovat QR kód směřující na vaši vlastní doménu (např. `attestation.jerela.eu`) s chybou `Scanned invalid QR code`.
2. **Oficiální server** naopak striktně vyžaduje, aby k němu přistupovala aplikace se jménem `app.attestation.auditor` podepsaná oficiálními klíči projektu GrapheneOS. Jakoukoliv vaši zkompilovanou aplikaci odmítne s chybou `invalid Auditor app package name` nebo `signature verification failed`.

Jelikož je navíc oficiální aplikace v GrapheneOS předinstalována jako systémová, nemůžete vaši vlastní zkompilovanou aplikaci pojmenovat stejně (`app.attestation.auditor`), protože by došlo ke konfliktu podpisů při instalaci.

## Řešení

### Krok 1: Úprava a kompilace vlastní Android aplikace
Musíte si zkompilovat vlastní verzi aplikace Auditor.
1. V souboru `app/build.gradle.kts` upravte `applicationId` na vlastní hodnotu, například:
   ```kotlin
   defaultConfig {
       applicationId = "app.attestation.auditor.custom"
       // ...
   }
   ```
2. Ponechte pro debugovací build příponu (případně odkomentujte):
   ```kotlin
   getByName("debug") {
       applicationIdSuffix = ".debug"
   }
   ```
3. Zkompilujte aplikaci a nainstalujte ji do telefonu přes `adb install` (nyní by se měla úspěšně nainstalovat vedle původní oficiální aplikace).
   Zkompilovaná aplikace bude mít název **`app.attestation.auditor.custom.debug`**.

### Krok 2: Úprava bezpečnostních pravidel na vašem Serveru
Nyní musíte donutit váš AttestationServer, aby plně důvěřoval vaší nové aplikaci a jejímu podpisu.

1. **Zjištění otisku vašeho podpisu (debug.keystore):**
   Spusťte na počítači nástroj `keytool` a zjistěte SHA-256 otisk vašeho klíče, kterým jste aplikaci podepsali:
   `keytool -list -v -keystore ~/.android/debug.keystore -storepass android`
   (Otisk očistěte od dvojteček, např. `0CA13FBE036E750674E8FA...`)

2. **Úprava AttestationProtocol.java:**
   Otevřete `src/main/java/app/attestation/server/AttestationProtocol.java` a najděte konstanty definující jméno aplikace a podpisy (kolem řádku 163-171). Původní oficiální údaje kompletně přepište na vaše:
   ```java
   private static final String AUDITOR_APP_PACKAGE_NAME_RELEASE = "app.attestation.auditor.custom.debug";
   private static final String AUDITOR_APP_PACKAGE_NAME_PLAY = "app.attestation.auditor.custom.debug";
   private static final String AUDITOR_APP_PACKAGE_NAME_DEBUG = "app.attestation.auditor.custom.debug";

   private static final String AUDITOR_APP_SIGNATURE_DIGEST_RELEASE = "VAS_SHA256_OTISK_BEZ_DVOJTECEK";
   private static final String AUDITOR_APP_SIGNATURE_DIGEST_PLAY = "VAS_SHA256_OTISK_BEZ_DVOJTECEK";
   private static final String AUDITOR_APP_SIGNATURE_DIGEST_DEBUG = "VAS_SHA256_OTISK_BEZ_DVOJTECEK";
   ```
   *(Tímto krokem maximalizujete bezpečnost – váš server komunikuje jen s vaší konkrétní aplikací).*

3. **Přidání otisku nového hardwaru (pokud máte nový nepodporovaný telefon, např. Pixel 10):**
   V `AttestationProtocol.java` nezapomeňte přidat root certifikát otisku vašeho konkrétního zařízení, pokud tam ještě není (přes logování výjimek zachytíte `unrecognized certificate fingerprint`, který pak přidáte do whitelistu).

### Krok 3: Sestavení a nasazení serveru
Po uložení kódu zkompilujte server:
```bash
./gradlew clean build -x test
```
Zkopírujte novou binárku na produkční místo a restartujte službu:
```bash
systemctl stop attestation
cp -r build/libs/* /cesta/k/deploy/
systemctl start attestation
```

### Krok 4: Zákaz veřejné registrace (Volitelné, ale doporučené pro osobní server)
Pokud nechcete, aby si na vašem serveru kdokoli mohl vytvořit účet (otevřená registrace je ve výchozím stavu zapnutá), můžete zablokovat příslušný koncový bod přímo na úrovni Nginx, aniž byste museli zasahovat do kódu serveru.

Otevřete konfiguraci vašeho Nginxu (např. `/etc/nginx/nginx.conf` nebo příslušný soubor v `sites-available`) a do bloku `server` před směrování na `/api/` přidejte blokaci:

```nginx
        location = /api/create-account {
            return 403;
        }
        
        location ^~ /api/ {
            proxy_pass http://backend;
        }
```
Poté Nginx restartujte: `systemctl reload nginx`. Od této chvíle server na pokus o vytvoření účtu odpoví chybou 403 Forbidden.

### Krok 5: Spárování
1. Přihlaste se na svůj účet ve webovém rozhraní vašeho AttestationServeru.
2. Spusťte na telefonu **svoji** custom kompilaci Auditoru (ne tu oficiální).
3. Naskenujte QR kód. Na telefonu by se měla zobrazit informace o úspěšném zpracování (případně success).
4. **Důležité:** Obnovte stránku webového rozhraní (F5). Spárovaný telefon se objeví v sekci zařízení.
