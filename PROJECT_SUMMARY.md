# BankApp Frontend - Podsumowanie projektu

## Czym jest BankApp

Open-source projekt bankowy z passwordless auth (email + passkeys). Backend auth działa, design jest w Figmie, API udokumentowane. Zadanie: zbudować frontend dla systemu autentykacji.

- Repo frontend: https://github.com/BankApp-project/bank-app-frontend
- Repo auth backend: https://github.com/BankApp-project/auth
- Repo bankapp backend: https://github.com/bankapp-project/bankapp-backend
- Live demo (plain JS): https://auth.bankapp.online/
- Swagger: https://auth.bankapp.online/api/swagger-ui/index.html
- Figma: https://www.figma.com/design/F9xTgzUobrmPqBcIe6wgQo/BankApp-Design?node-id=2328-819&p=f

---

## Backend auth - tech stack

- Java 25 + Spring Boot 3.5
- PostgreSQL + Redis (OTP) + Flyway
- RabbitMQ (email notifications)
- WebAuthn4J (passkeys/FIDO2)
- Docker / Docker Compose
- Uruchomienie lokalne: `docker compose up -d`, API pod `http://localhost:8080/api`

---

## API - endpointy

### User Verification

| Endpoint | Metoda | Opis |
|---|---|---|
| `/verification/initiate/email` | POST | Wysyla OTP na podany email. Async - zwraca 202 Accepted. |
| `/verification/complete/email` | POST | Weryfikuje OTP. Zwraca typ dalszego flow: `login` lub `registration`. |

**`POST /verification/initiate/email`**
- Request: `{ "email": "user@example.com" }`
- 202: Accepted (email z OTP wyslany w tle)
- 400: Nieprawidlowy format email

**`POST /verification/complete/email`**
- Request: `{ "email": "user@example.com", "otpValue": "..." }`
- 200: OTP poprawny. Response zawiera:
  - `type`: `"login"` lub `"registration"` - determinuje dalszy flow
  - Dla `login`: `loginOptions` (do `navigator.credentials.get()`) + `sessionId`
  - Dla `registration`: `registrationOptions` (do `navigator.credentials.create()`) + `sessionId`
- 400: Brakujace/nieprawidlowe dane
- 401: OTP nieprawidlowy, wygasly lub nie pasuje do email

### User Authentication (login)

| Endpoint | Metoda | Opis |
|---|---|---|
| `/authentication/initiate` | GET | Happy path - returning user na znanym urzadzeniu (session cookie). Zwraca `loginOptions`. |
| `/authentication/complete` | POST | Finalizuje login passkey. Zwraca tokeny. |

**`GET /authentication/initiate`**
- Brak parametrow (opiera sie na session cookie)
- 200: Zwraca `loginOptions` + `sessionId`
  ```json
  {
    "loginOptions": {
      "challenge": "string",
      "timeout": 0,
      "rpId": "string",
      "allowCredentials": [{ "type": "string", "id": "string", "transports": ["usb"] }],
      "userVerification": "required",
      "extensions": {}
    },
    "sessionId": "a1b2c3d4-e5f6-7890-1234-567890abcdef"
  }
  ```
- 401: Uzytkownik nierozpoznany / brak aktywnej sesji

**`POST /authentication/complete`**
- Request:
  ```json
  {
    "sessionId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
    "AuthenticationResponseJSON": "string",
    "credentialId": "01020304-0506-0708-090a-0b0c0d0e0f10"
  }
  ```
- 200: Sukces, zwraca `{ "accessToken": "string", "refreshToken": "string" }`
- 400: Brakujace wymagane pola lub nieprawidlowy format
- 401: Nieprawidlowy podpis, wygasla sesja, challenge mismatch

### User Registration

| Endpoint | Metoda | Opis |
|---|---|---|
| `/registration/complete` | POST | Rejestracja nowego usera z passkey. Zwraca tokeny. |

**`POST /registration/complete`**
- Request:
  ```json
  {
    "sessionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "RegistrationResponseJSON": "string"
  }
  ```
- 200: Konto utworzone, zwraca `{ "accessToken": "string", "refreshToken": "string" }`
- 400: Nieprawidlowa sesja, bledny request, lub weryfikacja passkey nieudana

---

## Schematy danych (ze Swaggera)

```
InitiateVerificationRequest {
  email: string (email)
}

CompleteVerificationRequest {
  email: string
  otpValue: string
}

LoginResponseDto {
  type: string           // "login" | "registration"
  loginOptions: object   // PublicKeyCredentialRequestOptions
  sessionId: string
}

RegistrationResponseDto {
  registrationOptions: object  // PublicKeyCredentialCreationOptions
  sessionId: string
}
```

### Typy WebAuthn

```
PublicKeyCredentialRequestOptions {
  challenge: string (wymagany)
  timeout: integer
  rpId: string
  allowCredentials: [PublicKeyCredentialDescriptor]
  userVerification: string
  extensions: object
}

PublicKeyCredentialCreationOptions {
  rp: object                    // relying party
  user: object                  // PublicKeyCredentialUserEntity
  challenge: string (wymagany)
  pubKeyCredParams: [PublicKeyCredentialParameters]
  timeout: integer
  excludeCredentials: [PublicKeyCredentialDescriptor]
  authenticatorSelection: AuthenticatorSelectionCriteria
  attestation: string
  extensions: object
}

PublicKeyCredentialDescriptor {
  type: string
  id: string (wymagany)
  transports: [string]
}

PublicKeyCredentialParameters {
  type: string
  alg: integer (wymagany)
}

PublicKeyCredentialRpEntity {
  id: string
  name: string
}

PublicKeyCredentialUserEntity {
  id: string (wymagany)
  name: string
  displayName: string
}

AuthenticatorSelectionCriteria {
  authenticatorAttachment: string
  requireResidentKey: boolean
  userVerification: string
}
```

---

## Flow uzytkownika

### Flow 1: Nowy uzytkownik (rejestracja)

```
[Email input] --> POST /verification/initiate/email
                        |
                  (email z OTP)
                        |
[OTP input]   --> POST /verification/complete/email
                        |
                  type: "registration"
                  registrationOptions + sessionId
                        |
              navigator.credentials.create(registrationOptions)
                        |
[Passkey]     --> POST /registration/complete
                        |
                  accessToken + refreshToken
                        |
                  [Sukces]
```

### Flow 2: Returning user - nowe urzadzenie

```
[Email input] --> POST /verification/initiate/email
                        |
                  (email z OTP)
                        |
[OTP input]   --> POST /verification/complete/email
                        |
                  type: "login"
                  loginOptions + sessionId
                        |
              navigator.credentials.get(loginOptions)
                        |
[Passkey]     --> POST /authentication/complete
                        |
                  accessToken + refreshToken
                        |
                  [Sukces]
```

### Flow 3: Returning user - znane urzadzenie (happy path)

```
(automatycznie, session cookie)
              --> GET /authentication/initiate
                        |
                  loginOptions
                        |
              navigator.credentials.get(loginOptions)
                        |
[Passkey]     --> POST /authentication/complete
                        |
                  accessToken + refreshToken
                        |
                  [Sukces]
```

---

## Ekrany z Figmy

Design obejmuje wersje desktop + mobile/responsive.

### Desktop (ekrany 5-10)

| Nr | Ekran | Opis |
|---|---|---|
| 5 | Welcome / Landing | Logo "BankApp", ikona banku, przycisk "Get started" |
| 6 | Email verification | Pola na email, awatary uzytkownikow (P, K) |
| 7 | Code confirmation | 6-cyfrowy OTP input, awatary |
| 8 | Authentication successful | Checkmark, komunikat sukcesu |
| 9 | Email verification (clean) | Wersja bez awatarow, minimalistyczna |
| 10 | Code confirmation (clean) | 6-cyfrowy OTP, niebieski przycisk "Verify", link "Didn't receive code?" |
| 11 | Email verification (wariant) | Kolejny wariant formularza email |

### Mobile (ekrany 1-4)

| Nr | Ekran | Opis |
|---|---|---|
| 1 | Welcome | Logo BankApp, krok "Step 1: Welcome" |
| 2 | Email verification | Formularz email, "Step 2: Verification" |
| 3 | Code confirmation | 6-cyfrowy OTP z czerwonym stanem (error?), "Step 3: Confirmation" |
| 4 | Authentication successful | Checkmark |

### Uwagi do Figmy

- Figma jest oznaczona jako **niekompletna** (moze byc jeszcze rozbudowana)
- Widac step indicator (Step 1, 2, 3) na mobile
- Sa warianty z awatarami i bez - do wyjasnienia ktore sa finalne
- Ekran 3 (mobile) sugeruje error state dla OTP (czerwone pola)
- Brak widocznych ekranow dla: happy path (passkey bez email), error states (serwer nieosiagalny, timeout), loading states

---

## Live demo

Obecna implementacja (plain JS) po udanej autentykacji pokazuje:
- Gradient tlo (fioletowo-niebieskie)
- Karta "Welcome Back!" z komunikatem "You're successfully authenticated and ready to go!"

---

## Wymagania

- Desktop + mobile/responsive
- Integracja z API (passkeys + email auth)
- Framework FE: do wyboru (React/Vue/Svelte/inne)
- Deadline: ~miesiac
- Hosting: wlasny lub dostarczony + domena bankapp.online
- CI/CD: pomoc ze strony backendu

---

## Znane ograniczenia backendu

- Problemy z kompatybilnoscia iOS WebAuthn
- JWT token implementation jeszcze w toku
- Uproszczona konfiguracja WebAuthn (wymaga hardening na produkcji)

---

## Otwarte pytania

- [ ] Jaki framework FE? (React/Vue/Svelte/inne)
- [ ] Ktore warianty z Figmy sa finalne? (z awatarami vs bez)
- [ ] Czy beda ekrany error states w Figmie, czy implementujemy sami?
- [ ] Happy path - jak backend rozpoznaje znane urzadzenie? (session cookie - jak dlugo zyje?)
- [ ] Co po uzyskaniu tokenow? (redirect? standalone app?)
- [ ] CORS - frontend na jakiej domenie/subdomenie?
- [ ] CI/CD - GitHub Actions? Co juz istnieje?
- [ ] Brakujace ekrany w Figmie - co jeszcze bedzie dodane?
