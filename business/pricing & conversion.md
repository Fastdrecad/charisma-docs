### **Ažurirana Strategija Rasta i Monetizacije (Verzija 2.0)**

#### **Osnovna Filozofija: "Mašina za Rast"**

Naš cilj nije samo da naplaćujemo funkcije. Naš cilj je da izgradimo **ekosistem** koji:

1.  **Privlači** korisnike bez otpora (niska frikcija).
2.  **Demonstrira** ogromnu vrednost kroz korišćenje.
3.  **Nudi** fleksibilne puteve ka monetizaciji.
4.  **Zadržava** korisnike kroz stalnu vrednost i gamifikaciju.

Ovo postižemo kroz **četiri ključna stuba:**

---

#### **Stub 1: Hibridni Monetizacioni Motor (Osnova)**

Ovo ostaje naša osnova, ali sa jasnijim definicijama.

- **A. Besplatni Nivo (Free Tier):** Motor za **akviziciju**.
  - **Dobija:** 50 kredita pri registraciji + 10 besplatnih kredita svakog dana.
  - **Cilj:** Maksimalno povećati bazu korisnika (MAU) i organski rast.
- **B. Krediti (Pay-As-You-Go):** Motor za **fleksibilnu monetizaciju**.
  - **Dobija:** Mogućnost kupovine paketa kredita.
  - **Cilj:** Monetizovati "Explorers" i povremene korisnike koji nisu spremni za pretplatu.
- **C. Nedeljna Pretplata (Premium):** Motor za **predvidljiv prihod (MRR)**.
  - **Dobija:** Neograničeno korišćenje i ekskluzivne "power" funkcije.
  - **Cilj:** Stvoriti stabilan, visokoprofitabilan prihod, što je ključno za valuaciju kompanije.

---

#### **Stub 2: "Vrednost-Prvo" Akvizicija (Novi Trial Model)**

Odbacujemo klasični, vremenski ograničen probni period.

- **Novi Model:** **"Prve 3 Premium Sesije su Besplatne"**
- **Kako Radi:** Svaki novi korisnik može da iskoristi našu najmoćniju, **premium-only "Session" funkciju** tri puta, potpuno besplatno, **bez unosa kreditne kartice.**
- **Zašto je Ovo Bolje?**
  - **Nema Otpora:** Uklanja najveću prepreku – strah od zaboravljanja otkazivanja.
  - **Demonstrira Maksimalnu Vrednost:** Odmah mu pokazujemo naš najimpresivniji alat, stvarajući snažan "Aha!" momenat.
  - **Kvalifikuje Korisnike:** Korisnici koji iskoriste sve 3 sesije su naši najangažovaniji i najverovatniji budući pretplatnici.

---

#### **Stub 3: "Power Advantage" Premium Ponuda**

Premium nije samo "više", već **"bolje"**.

- **A. Ekskluzivne Funkcije:**
  - **`Session Feature`:** Korišćenje analize screenshota je **isključivo za premium korisnike** (nakon što potroše 3 besplatne probne sesije).
  - **`Bookmarks`:** Čuvanje najboljih openera je premium funkcija.
- **B. Superioran AI Kvalitet:**
  - **Free/Credit Korisnici:** Dobijaju odgovore od brzih i isplativih modela (npr. `gpt-4o-mini`, `gemini-flash`). Rezultati su dobri.
  - **Premium Korisnici:** Dobijaju odgovore od najnaprednijih modela koje imamo (npr. `claude-4.5-sonnet`, `gpt-4o`, `gemini 3.0`). Rezultati su **izvanredni**. Ovo je opipljiva razlika u kvalitetu koju korisnik direktno oseća.
- **C. Neograničeno Korišćenje:** Nema brige o kreditima.

---

#### **Stub 4: Gamifikovano Angažovanje i Segmentacija**

Pretvaramo korišćenje aplikacije u igru.

- **A. Segmentacija Korisnika (MVP):**
  - **Implementacija:** Tokom onboarding-a, postavljamo pitanje: "Koji je tvoj glavni cilj?". Na osnovu odgovora, dodeljujemo im segment (`Explorer`, `Dater`, `Coach`).
  - **Cilj:** Prikupljamo podatke od prvog dana da bismo u budućnosti mogli da personalizujemo ponude.
- **B. "Credit Sinks" (Roadmap V1.1+):**
  - **Implementacija:** Uvodimo mikro-transakcije unutar aplikacije.
    - _"Poboljšaj ovaj opener sa više `charming` stila"_ (Cena: 2 kredita).
    - _"Generiši još 3 varijacije na ovu poruku"_ (Cena: 3 kredita).
  - **Cilj:** Povećati potrošnju besplatnih dnevnih kredita, održati korisnike aktivnim i stvoriti stalnu potrebu za dopunom ili pretplatom.

---

### **Tehničke Implikacije (Ažurirano)**

1.  **Prisma Šema (`User` model):**

    ```prisma
    model User {
      // ...
      credits         Int      @default(50)
      isPremium       Boolean  @default(false) @map("is_premium")
      subscriptionEnd DateTime? @map("subscription_end")

      // Za "Vrednost-Prvo" Trial
      freeSessionsUsed Int @default(0) @map("free_sessions_used")

      // Za segmentaciju
      userSegment     String?  @map("user_segment") // 'explorer', 'dater', 'coach'
    }
    ```

2.  **Servisi:**
    - **`CreditService`:** Ostaje kako je planirano.
    - **`SubscriptionService`:** Novi servis za upravljanje statusom pretplate (integracija sa RevenueCat/App Store/Google Play).
    - **`SessionService`:** `createSession` metoda sada mora da proveri `user.isPremium` ili `user.freeSessionsUsed < 3`. Ako nijedan uslov nije ispunjen, baca grešku `403 Forbidden`.
3.  **Middleware:**
    - **`premiumOnly` middleware:** Ostaje za `bookmark` rute. Sada se primenjuje i na **sve `session` rute**, osim na one koje proveravaju status.
4.  **AI Klijent (`aiStrategy`):**
    - Mora biti svestan statusa korisnika. `getProviderForUseCase` će primati `isPremium` kao argument i na osnovu toga birati `gemini-flash` ili `claude-sonnet`.

---

### **Ažurirani Korisnički Put (Conversion Funnel)**

1.  **Novi Korisnik:** Registruje se -> Odgovara na pitanje o segmentaciji -> Dobija 50 kredita i informaciju o 3 besplatne premium sesije.
2.  **Korišćenje:**
    - **Magic Openers:** Troši kredite. Kada ostane bez njih -> vidi opciju "Kupi Kredite" ILI "Postani Premium za neograničeno".
    - **Sessions:** Koristi svoje 3 besplatne sesije. Nakon treće -> vidi ekran "Otključaj neograničene sesije sa Premiumom".
    - **Bookmarks:** Pokuša da sačuva opener -> vidi "Otključaj sa Premiumom" ekran.
3.  **Monetizacija:** Korisnik ima **jasan izbor** na osnovu svog ponašanja: ako je povremeni, kupiće kredite; ako je ozbiljan, uzeće pretplatu.

Ova strategija je naša mapa za uspeh. Fleksibilna je, moćna i optimizovana za rast i profitabilnost. Možemo nastaviti sa implementacijom.
