# Cómo construimos un Trading Platform completo con IA: Arquitectura real, decisiones técnicas y lecciones

> **TL;DR:** Construimos una plataforma de trading algorítmico completa: backtesting engine, señales IA, ejecución MT5 y gestión de riesgo. En este post compartimos la arquitectura real, las decisiones de stack, los errores que cometimos y cómo puedes construir algo similar en semanas, no meses.

---

## Por qué construir un trading platform (y por qué es tan difícil)

La mayoría de personas que intentan automatizar trading terminan con un script de Python que descarga datos, calcula una media móvil y envía emails. Eso no es un trading platform. Es un juguete.

Un trading platform real necesita:

- **Backtesting engine** — probar estrategias contra años de datos históricos
- **Ejecución automatizada** — enviar órdenes a un broker en tiempo real
- **Gestión de riesgo** — position sizing, stop loss, max drawdown
- **Señales de entrada/salida** — el cerebro algorítmico
- **Monitoring** — saber qué está haciendo tu dinero mientras duermes
- **Analytics** — 20+ métricas de rendimiento (Sharpe, Sortino, Calmar, etc.)

Hacer todo eso bien es un proyecto de ingeniería serio. Y hacerlo con señales generadas por IA lo es aún más.

Nosotros lo construimos. Y ahora te contamos cómo.

---

## Arquitectura elegida

### Stack técnico

```
Frontend:    Next.js 14 + TypeScript + Tailwind CSS + Lightweight Charts
Backend:     Node.js + Express + WebSocket
Database:    PostgreSQL (Neon) + Redis para caché
IA:          Python + OpenAI API + modelos custom
Trading:     MetaTrader 5 + Expert Advisors (MQL5)
Deploy:      Vercel (frontend) + Railway (backend)
```

### Diagrama de flujo simplificado

```
Datos históricos → Backtesting Engine → Estrategia validada
                                            ↓
Datos en tiempo real → Motor de Señales IA → Señal de trading
                                                    ↓
                                        Gestor de Riesgo (OK?)
                                                    ↓
                                        Ejecución MT5 (orden)
                                                    ↓
                                        Monitoring Dashboard
```

### Por qué este stack

**Next.js para el frontend**: Necesitábamos gráficos de velas en tiempo real, actualización sin recarga y SEO para las páginas públicas. Next.js con React Server Components nos dio todo.

**Python para la IA**: No hay discusión. El ecosistema de ML (pandas, numpy, scikit-learn, PyTorch) sigue siendo muy superior. Las señales de trading van en Python, el resto en TypeScript.

**PostgreSQL para datos financieros**: Los datos de mercado tienen estructura clara (OHLCV, timestamps, symbols). SQL relacional es perfecto para esto. Redis para las señales en tiempo real que necesitan latencia baja.

**MT5 para ejecución**: MetaTrader 5 domina el mercado retail de FX. Los Expert Advisors (EAs) permiten ejecución automatizada con control total. No reinventamos la rueda de ejecución.

---

## El Backtesting Engine: el corazón del sistema

El backtester es el componente más crítico. Si tu backtesting no es fiel a la realidad, tus estrategias fracasarán en producción. Sin excepciones.

### Qué simula nuestro backtester

- **Datos OHLCV** de 10+ años, múltiples símbolos y timeframes
- **Spread y comisiones** realistas por broker
- **Slippage** — el precio de ejecución no es el precio del gráfico
- **Position sizing** — fijo, porcentaje de equity, Kelly criterion
- **Margin calls** — si tu estrategia usa apalancamiento, el backtester debe saber cuándo te liquidan

### Las 20+ métricas que calculamos

| Métrica | Qué mide | Buen valor |
|---------|----------|------------|
| Total Return | Rentabilidad total | > 30% anual |
| Sharpe Ratio | Retorno ajustado por riesgo | > 1.5 |
| Sortino Ratio | Solo riesgo negativo | > 2.0 |
| Max Drawdown | Mayor pérdida desde pico | < 20% |
| Calmar Ratio | Return / Max Drawdown | > 2.0 |
| Win Rate | % trades ganadores | > 55% |
| Profit Factor | Ganancias / Pérdidas | > 1.5 |
| Expectancy | Esperanza matemática por trade | > 0 |
| Recovery Factor | Profit / Max Drawdown | > 3 |
| Exposure | % del tiempo en mercado | variable |

### El error que nadie te dice sobre backtesting

**Look-ahead bias**: usar datos del futuro en el pasado. Sucede cuando normalizas datos usando todo el dataset antes de dividir en train/test. O cuando tus indicadores usan barras que aún no existen.

**Overfitting**: optimizar tanto que la estrategia solo funciona en datos históricos. En nuestro sistema, usamos Walk-Forward Analysis obligatorio: optimizas en ventana A, validas en ventana B (datos nunca vistos).

**Survivorship bias**: probar solo contra símbolos que aún existen. Si un par de divisas fue deslistado en 2020, no estará en tu dataset, y tu backtester será demasiado optimista.

---

## Señales de IA: donde la magia sucede

Un trading platform sin señales es solo un dashboard bonito. Las señales son el cerebro.

### Nuestro enfoque: ensemble de indicadores + ML

No confiamos en un solo indicador. Usamos un ensemble:

1. **ICT (Inner Circle Trader) concepts** — order blocks, fair value gaps, liquidity sweeps
2. **Indicadores clásicos** — RSI, MACD, ATR, Bollinger Bands, Volume Profile
3. **Modelos ML** — Random Forest para clasificación de dirección (sube/baja), LSTM para predicción de volatilidad

### Cómo generamos la señal final

```python
signals = {
    'ict': ict_score(order_blocks, fvgs, liquidity),  # -1 to 1
    'classic': indicator_score(rsi, macd, atr),       # -1 to 1
    'ml': model_predict(features),                     # -1 to 1
}
final = weighted_ensemble(signals, weights=[0.4, 0.3, 0.3])
if abs(final) > threshold and risk_check():
    execute_signal(final)
```

### El problema de latencia

En scalping (trades de segundos/minutos), la latencia mata. Si tu señal tarda 500ms en generarse y 200ms en ejecutar, el mercado se movió.

Para estrategias swing (horas/días), lo que importa es la calidad de la señal, no la latencia. Nosotros apuntamos a swing + intraday, no scalping.

---

## Gestión de riesgo: lo que separa a los vivos de los muertos

La gestión de riesgo NO es opcional. Es lo que hace la diferencia entre un trader que sobrevive 5 años y uno que explota la cuenta en 3 meses.

### Nuestras reglas de riesgo

1. **Nunca arriesgar más del 2% del capital por trade**
2. **Max drawdown diario del 5%** — si lo alcanzas, pará hoy
3. **Max posiciones abiertas: 3** — la concentración excesiva mata
4. **Stop loss obligatorio** — cada trade sin SL es una ruleta rusa
5. **Take profit basado en ATR** — fixed TP es para amateurs

### Position sizing con Kelly Criterion

```
Kelly % = W - (1 - W) / R

Donde:
W = win rate histórica
R = ratio promedio ganancia/pérdida

Ejemplo: Win rate 60%, R = 2:1
Kelly % = 0.60 - 0.40/2 = 0.40 = 40%

Usamos Half-Kelly (20%) para ser conservadores.
```

---

## Lecciones aprendidas (las que dolió aprender)

### 1. El backtesting perfecto no existe
Siempre hay diferencias entre backtesting y producción. El mercado real tiene slippage, rechazos de órdenes y gaps que el backtester no modela perfectamente. Espera un 20-30% de degradación en producción.

### 2. La IA no es una bola de cristal
Los modelos ML pueden encontrar patrones en datos históricos que no se repiten. El mercado cambia. Lo que funcionó en 2023 puede no funcionar en 2026. Re-entrena modelos regularmente.

### 3. La infraestructura importa más de lo que crees
Un trading platform necesita uptime. Si tu servidor se cae durante un evento de volatilidad (NFP, FOMC), puedes perder dinero real. Usa monitoring, alertas y failover.

### 4. Empieza simple, itera rápido
Nuestra primera versión solo tenía backtesting + un indicador. Nada de IA, nada de MT5. Validamos el concepto, después añadimos complejidad. No construyas el sistema perfecto desde el día 1.

### 5. El mayor riesgo es el emocional
Incluso con un sistema automatizado, la tentación de "intervenir" es real. Si tu sistema dice SELL y tú crees que debería ser BUY, ¿a quién obedeces? La disciplina es el edge.

---

## Cómo construir tu propio trading platform

Si quieres construir algo similar, aquí va el roadmap:

### Fase 1: Backtesting (2-3 semanas)
- Descarga datos históricos (Dukascopy, TrueFX, o APIs de pago)
- Construye un backtester básico en Python (pandas + vectorizado)
- Implementa 5-10 métricas de rendimiento
- Valida 2-3 estrategias simples

### Fase 2: Señales (2-3 semanas)
- Añade indicadores técnicos (ta-lib o pandas-ta)
- Implementa al menos un modelo ML (Random Forest es buen punto de partida)
- Crea el sistema de ensemble

### Fase 3: Frontend (2-3 semanas)
- Next.js + Lightweight Charts para gráficos
- Dashboard con métricas en tiempo real
- Sistema de alertas (email, Telegram)

### Fase 4: Ejecución (3-4 semanas)
- MT5 Expert Advisor para ejecución
- Gestión de órdenes (market, limit, stop)
- Reconexión automática si se pierde conexión

### Fase 5: Producción (ongoing)
- Monitoring 24/7
- Re-entrenamiento de modelos mensual
- Optimización continua

**Total estimado: 10-16 semanas con un equipo de 2-3 personas.**

---

## ¿Necesitas ayuda para construir el tuyo?

En ZOO hemos construido trading platforms completos con IA, backtesting engines y sistemas de ejecución automatizada. Si estás construyendo algo en este espacio y necesitas ingeniería especializada, hablemos.

No somos una agencia genérica. Somos ingenieros que entienden trading, IA y sistemas de producción.

**Contacto:** crazycompanyincmail@gmail.com

---

*¿Te resultó útil este post? Compártelo con alguien que esté construyendo en el espacio de trading algorítmico.*
