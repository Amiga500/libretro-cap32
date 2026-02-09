# Guida Compilazione per Miyoo Mini Plus
## Usando Docker union-miyoomini-toolchain

Questa guida spiega come compilare libretro-cap32 ottimizzato per Miyoo Mini Plus usando il container Docker union-miyoomini-toolchain.

---

## 📋 Prerequisiti

Assicurati di essere dentro il container Docker:
```bash
# Verifica di essere nel container
echo $TOOLCHAIN
# Dovrebbe mostrare qualcosa tipo /opt/miyoomini-toolchain

# Il toolchain dovrebbe essere in:
ls /opt/miyoomini-toolchain/bin/arm-linux-gnueabihf-gcc
```

---

## 🚀 Compilazione Rapida (Metodo Consigliato)

### Opzione 1: Usando lo script automatico con PATH configurato

```bash
# Aggiungi il toolchain al PATH (importante!)
export PATH=/opt/miyoomini-toolchain/bin:$PATH

# Metodo più semplice - usa lo script build_arm.sh
./build_arm.sh miyoomini

# Oppure con più core per compilazione più veloce
./build_arm.sh miyoomini 8
```

### Opzione 2: Compilazione diretta (senza script)

```bash
# Aggiungi il toolchain al PATH
export PATH=/opt/miyoomini-toolchain/bin:$PATH

# Compila
make platform=miyoomini clean
make platform=miyoomini -j4
```

**NOTA IMPORTANTE**: Il Makefile ora rileva automaticamente il toolchain in `/opt/miyoomini-toolchain/bin/`. Se il toolchain è in un'altra posizione, aggiungi quella directory al PATH.

Lo script farà automaticamente:
1. Clean della build precedente
2. Compilazione con ottimizzazioni ARM Cortex-A7 + NEON
3. Verifica del core generato
4. Mostra istruzioni per il deployment

---

## 🔧 Compilazione Manuale (Metodo Alternativo)

Se il toolchain è in una posizione diversa o vuoi specificare manualmente:

### Passo 1: Configura il PATH o specifica il compilatore

**Metodo A - Usa PATH (consigliato):**
```bash
export PATH=/opt/miyoomini-toolchain/bin:$PATH
```

**Metodo B - Specifica manualmente (solo se necessario):**
```bash
# Se il toolchain è in una posizione diversa
export MIYOO_TOOLCHAIN=/percorso/al/toolchain/bin
make platform=miyoomini CC=$MIYOO_TOOLCHAIN/arm-linux-gnueabihf-gcc \
     CXX=$MIYOO_TOOLCHAIN/arm-linux-gnueabihf-g++ \
     AR=$MIYOO_TOOLCHAIN/arm-linux-gnueabihf-ar
```

### Passo 2: Clean (opzionale ma consigliato)
```bash
make platform=miyoomini clean
```

### Passo 3: Compilazione
```bash
# Compilazione standard (4 core)
make platform=miyoomini -j4

# Compilazione più veloce (8 core se disponibili)
make platform=miyoomini -j8

# Compilazione con log dettagliati (per debug)
make platform=miyoomini V=1
```

### Passo 4: Verifica
```bash
# Verifica che il file sia stato creato
ls -lh cap32_libretro.so

# Controlla che sia un binario ARM
file cap32_libretro.so
# Output atteso: "ELF 32-bit LSB shared object, ARM, EABI5..."

# Controlla la dimensione (dovrebbe essere ~500KB-1MB dopo strip)
du -h cap32_libretro.so
```

---

## 📦 Ottimizzazioni Incluse

Quando compili con `platform=miyoomini`, ottieni automaticamente:

✅ **8-bit Color Mode (8bpp)** - Rendering RGB565 ad alte prestazioni  
✅ **ARM NEON SIMD** - Palette lookup vettorizzato (8 pixel alla volta)  
✅ **Screen Crop** - Auto-crop per display 320×240  
✅ **CPC 464 Default** - Modello leggero (64KB vs 128KB)  
✅ **Frameskip Automatico** - Skip dinamico dei frame quando serve  
✅ **Compiler Flags Aggressivi** - -O3 -flto -ffast-math -mfpu=neon-vfpv4  

**Gain FPS stimato: +40-58 FPS → Target 60 FPS** 🎯

---

## 📤 Deployment su Miyoo Mini Plus

### Metodo 1: Via SCP (rete)
```bash
# Copia il core sulla Miyoo (sostituisci IP con quello del tuo dispositivo)
scp cap32_libretro.so root@192.168.1.XXX:/mnt/SDCARD/RetroArch/.retroarch/cores/

# Oppure con nome utente specifico
scp cap32_libretro.so miyoo@192.168.1.XXX:/mnt/SDCARD/RetroArch/.retroarch/cores/
```

### Metodo 2: Via SD Card
```bash
# Copia il file in una directory temporanea
cp cap32_libretro.so /tmp/

# Poi monta la SD card sul tuo PC e copia manualmente in:
# /RetroArch/.retroarch/cores/cap32_libretro.so
```

### Metodo 3: Via ADB (se configurato)
```bash
adb push cap32_libretro.so /mnt/SDCARD/RetroArch/.retroarch/cores/
```

---

## ⚙️ Configurazione RetroArch (Dopo l'Installazione)

Una volta copiato il core, configura RetroArch per massime prestazioni:

**Menu Rapido → Opzioni:**
- **Model**: `464` (più leggero del 6128)
- **Internal Resolution**: `8bit` (più veloce)
- **Crop Screen Borders**: `enabled` (meno pixel da processare)
- **Frameskip**: `auto` (skip automatico quando serve)
- **Floppy Sound**: `disabled` (salva +2-3% CPU)

---

## 🐛 Risoluzione Problemi

### Errore: "arm-linux-gnueabihf-gcc: command not found"

Il toolchain potrebbe non essere nel PATH. Soluzione:

```bash
# Verifica che il toolchain esista
ls /opt/miyoomini-toolchain/bin/arm-linux-gnueabihf-gcc

# Aggiungi al PATH
export PATH=/opt/miyoomini-toolchain/bin:$PATH

# Verifica che funzioni
which arm-linux-gnueabihf-gcc
# Dovrebbe mostrare: /opt/miyoomini-toolchain/bin/arm-linux-gnueabihf-gcc

# Ora compila
make platform=miyoomini clean
make platform=miyoomini -j4
```

### Errore: "cc: error: unrecognized command line option '-mfpu=neon-vfpv4'"

Questo errore significa che sta usando il compilatore di sistema (`cc` o `gcc`) invece del cross-compiler ARM. Soluzioni:

**Soluzione 1 - Aggiungi toolchain al PATH (CONSIGLIATO):**
```bash
export PATH=/opt/miyoomini-toolchain/bin:$PATH
make platform=miyoomini clean
make platform=miyoomini -j4
```

**Soluzione 2 - Unset variabili ambiente conflittuali:**
```bash
unset CC CXX AR
export PATH=/opt/miyoomini-toolchain/bin:$PATH
make platform=miyoomini clean
make platform=miyoomini -j4
```

**Soluzione 3 - Usa percorso assoluto:**
```bash
make platform=miyoomini clean
make platform=miyoomini -j4 \
     CC=/opt/miyoomini-toolchain/bin/arm-linux-gnueabihf-gcc \
     CXX=/opt/miyoomini-toolchain/bin/arm-linux-gnueabihf-g++ \
     AR=/opt/miyoomini-toolchain/bin/arm-linux-gnueabihf-ar
```

### Errore: "cc: warning: '-mcpu=' is deprecated"

Se vedi questo warning ma la compilazione continua, è normale per alcune versioni di GCC. Non è un errore critico.

### Il toolchain è in una posizione diversa

Se il tuo toolchain non è in `/opt/miyoomini-toolchain/bin/`:

```bash
# Trova il compilatore
find /opt /usr/local -name "arm-linux-gnueabihf-gcc" 2>/dev/null

# Usa il percorso che trovi
export PATH=/percorso/trovato/bin:$PATH
make platform=miyoomini -j4
```

### Errore: "undefined reference to `vld1_u8'"

Le NEON intrinsics non sono abilitate. Assicurati che la compilazione usi `-mfpu=neon-vfpv4`:

```bash
# Ricompila con verbose per vedere i flag
make platform=miyoomini clean
make platform=miyoomini V=1 | grep mfpu

# Dovresti vedere: -mfpu=neon-vfpv4
```

### Il core è troppo grande (> 2MB)

Prova a ricompilare senza debug symbols:

```bash
make platform=miyoomini clean
make platform=miyoomini -j4
strip cap32_libretro.so
```

### Crash o performance basse

1. Verifica che il core sia ARM (non x86):
   ```bash
   file cap32_libretro.so
   # Deve dire "ARM"
   ```

2. Controlla la configurazione in RetroArch (deve essere 8bit, crop enabled)

3. Testa con una ROM semplice prima (es. un gioco BASIC)

---

## 🧪 Test Performance

Dopo l'installazione, testa con questi giochi per vedere i miglioramenti:

**Giochi Leggeri** (dovrebbero fare 60 FPS):
- Roland in the Caves
- Jet Set Willy
- Manic Miner

**Giochi Medi** (dovrebbero fare 60 FPS):
- Rick Dangerous
- Golden Axe
- Rainbow Islands

**Giochi Pesanti** (dovrebbero fare 55-60 FPS):
- Chase HQ
- Batman - The Movie
- RoboCop
- Operation Wolf

Abilita "Show Framerate" in RetroArch (Settings → Onscreen Display → Show Framerate) per monitorare gli FPS.

---

## 📊 Performance Attese

| Tipo Gioco | FPS Prima | FPS Dopo | Miglioramento |
|------------|-----------|----------|---------------|
| BASIC/Semplici | 30-45 | 60 | +33-100% |
| Medi | 25-35 | 60 | +71-140% |
| Pesanti | 18-25 | 55-60 | +140-233% |

---

## 💡 Tips Avanzati

### Compilazione con Log Performance
```bash
make platform=miyoomini LOG_PERFORMANCE=1 -j4
```

### Compilazione Debug (per sviluppatori)
```bash
make platform=miyoomini DEBUG=1 -j4
```

### Ricompilazione Veloce (solo file modificati)
```bash
# Non fare clean, compila solo ciò che è cambiato
make platform=miyoomini -j8
```

### Cross-Compilazione da Linux Host
Se non sei nel container ma vuoi compilare da Linux host:
```bash
# Installa il toolchain ARM
sudo apt-get install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf

# Poi compila
make platform=miyoomini -j4
```

---

## 📚 Ulteriori Informazioni

- **Guida Tecnica Completa**: Vedi `ARM_OPTIMIZATION.md`
- **Dettagli Implementazione**: Vedi `IMPLEMENTATION_SUMMARY.md`
- **Script Test FPS**: Usa `./test_fps.sh` per benchmark

---

## 🆘 Supporto

Se hai problemi:

1. Verifica di essere nel container Docker corretto
2. Controlla che il toolchain ARM sia nel PATH
3. Leggi i messaggi di errore completi
4. Prova prima con `make platform=unix` per vedere se compila in generale
5. Controlla che tutti i file sorgente siano presenti

Per ulteriore aiuto, apri un issue su GitHub con:
- Output completo dell'errore
- Versione del toolchain (`gcc --version`)
- Platform (`uname -a`)

---

## 🎉 Buona Compilazione!

Una volta compilato e installato il core, dovresti vedere miglioramenti significativi nelle performance. Goditi i tuoi giochi Amstrad CPC a 60 FPS sulla Miyoo Mini Plus! 🚀
