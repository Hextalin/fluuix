# Fluuix · Treasury Rail

Fluuix reúne la liquidez de un grupo empresario en una caja central con saldos por operativa. Desde la web, el tesorero puede mover fondos entre sociedades, registrar las condiciones de cada préstamo y conciliarlo con su transacción en Avalanche. El prototipo se construyó para [HackaTwin 2026](https://luma.com/cudf7cqz).

## Demo y evidencia on-chain

| Dato | Configuración de la demo |
| --- | --- |
| App desplegada | [Fluuix](https://fluuix.vercel.app/) |
| Red | Avalanche Fuji C-Chain, chain ID `43113`. El gas se paga con AVAX de prueba. |
| Vault | [`0xa0BA81F704C74b0607628e91771dA7AC5B4B11EE` en el explorador de Fuji](https://explorer-test.avax.network/c-chain/address/0xa0BA81F704C74b0607628e91771dA7AC5B4B11EE), bloque de despliegue `60690539`. [Código verificado de TreasuryVault](https://testnet.snowtrace.io/address/0xa0BA81F704C74b0607628e91771dA7AC5B4B11EE#code). |
| Token | [mARGt (`0x41bA39113a01B623779C3956F5852Dc9953bfF91`) en el explorador de Fuji](https://explorer-test.avax.network/c-chain/address/0x41bA39113a01B623779C3956F5852Dc9953bfF91), ERC-20 de prueba con 18 decimales. |
| Préstamo intercompany | [`0xc7f2d92f022d56b5868be55b4c7b16e13408402b2e7333253f07691c1e5c98fd` en el explorador de Fuji](https://explorer-test.avax.network/c-chain/tx/0xc7f2d92f022d56b5868be55b4c7b16e13408402b2e7333253f07691c1e5c98fd). Movió 4 mARGt de la operativa 1 a Caja central con una referencia `FLX:` de Fluuix. |

El prototipo usa mARGt en Fuji. Integrar una stablecoin oficial de Twin es un paso pendiente para cumplir la consigna del track Stablecoins.

### Demo en vivo

1. Entrá en `/mover` con una wallet con el rol `TESORERO` y AVAX de prueba. Firmá el acceso, elegí dos operativas con saldo suficiente en origen y registrá un préstamo. Confirmá la transacción en la wallet.
2. Volvé a `/dashboard`. El registro muestra el capital, la tasa y el enlace al hash en Snowtrace cuando el préstamo queda confirmado.
3. Para mostrar la automatización, prepará Caja central con 9,5 mARGt y Palermo con 0,5 mARGt. En `/dashboard`, elegí **Pico de tráfico y ventas** y **Escenario de pitch · 5 segundos**. Tocá **Evaluar tesorería** y dejá la pestaña abierta. Si dejás vencer el plazo de veto, el servidor revalida los saldos y ejecuta el préstamo de 1,5 mARGt al 5 % TNA. El tráfico, las ventas y el umbral de este escenario son simulados. La tasa queda registrada como condición del préstamo.

## Problema y modelo de negocio

Una cadena de locales, una distribuidora o un holding puede tener efectivo ocioso en una sociedad mientras otra necesita cubrir pagos. El tesorero suele mover esos fondos entre cuentas y luego conciliar por separado saldos, comprobantes y condiciones del préstamo.

El cliente inicial es el responsable de tesorería de un grupo PyME o mediano en Argentina con varias sociedades. El modelo propuesto es una suscripción mensual por operativa, con planes según volumen. El diferencial es que el equipo puede ver la caja consolidada, mover liquidez y recuperar el comprobante junto con las condiciones del préstamo desde un mismo flujo.

## Arquitectura y por qué Avalanche

- **Web y API.** Next.js y TypeScript sirven la interfaz y la API intercompany desde `web/`. En modo Fuji, la app lee el vault con viem y la wallet del tesorero firma el movimiento manual.
- **Vault en Fuji.** `TreasuryVault` mantiene los saldos por operativa. Cada préstamo mueve el saldo interno en el contrato y emite un evento con la referencia de la operación.
- **Registro en Postgres.** La API guarda capital, tasa y nota en Supabase. Antes de confirmar una operación verifica el emisor, el contrato, los argumentos y el evento de la transacción. La referencia `FLX:<uuid>` vincula el registro con el movimiento en Fuji.
- **Automatización.** El servidor aplica reservas y límites de liquidez y ofrece una ventana de veto antes de firmar con una cuenta dedicada de Fuji. En modo normal, el modelo de IA puede estimar el umbral preventivo. La política de tesorería determina el monto y la cuenta de origen.

Elegimos Avalanche C-Chain por la finalidad rápida de las transacciones y la compatibilidad con Solidity y wallets EVM. La finalidad importa en el flujo de préstamo porque el tesorero puede comprobar el saldo actualizado y conciliar el movimiento en la misma sesión. Fuji permite probar ese flujo con transacciones reales de testnet. [Documentación de Avalanche C-Chain](https://docs.avax.network/docs/primary-network)

## Instalación local

Usá Node.js 22 y npm. Ejecutá estos comandos desde la raíz del repositorio.

```sh
cd web
npm ci
npm run dev
```

Abrí [el dashboard local](http://localhost:3000/dashboard). Con `NEXT_PUBLIC_MODE=mock` (el valor por defecto) podés probar depósitos, préstamos, rebalanceo y pagos en lote con datos simulados en el navegador. Si ya tenés un `web/.env.local` configurado para Fuji, cambiá el modo a `mock` y reiniciá el servidor para usar el simulador.

Para correr las pruebas, ejecutá este comando desde `web/`.

```sh
node --experimental-strip-types --test scripts/treasury-rail.test.mjs scripts/treasury-automation.test.mjs
```

## Despliegue en Vercel con Fuji

1. Prepará el vault de Fuji con el token de prueba y al menos dos operativas activas. Configurá una wallet con el rol `TESORERO` y saldo de AVAX de prueba. Para desplegar o verificar el contrato, seguí la [guía técnica](web/TREASURY-RAIL.md#activar-intercompany-en-avalanche-fuji).
2. Creá el proyecto de Supabase y aplicá [la migración intercompany](web/db/001_intercompany.sql). Para habilitar la automatización, aplicá también [la segunda migración](web/db/002_treasury_automation.sql).
3. Importá el repositorio en Vercel con el preset **Next.js** y **Root Directory `web`**. Usá la [plantilla de entorno](web/.env.example) y configurá `NEXT_PUBLIC_MODE=chain`, las direcciones del vault y del token y el bloque de despliegue. En el servidor configurá `DATABASE_URL`, `INTERCOMPANY_SESSION_SECRET` y `APP_ORIGIN` con el dominio HTTPS de la app.
4. Para la automatización, configurá `TREASURY_AUTOMATION_VAULT` y guardá `TREASURY_AUTOMATION_PRIVATE_KEY` como secreto de servidor de una wallet dedicada a Fuji con el rol `TESORERO`. `OPENAI_API_KEY` es opcional para la evaluación en modo normal. Desplegá y probá el flujo de préstamo desde el dominio configurado. Para probar el mismo entorno localmente, usá `web/.env.local` y el origen local.

## Qué construimos durante HackaTwin

Sobre Next.js y las bibliotecas de código abierto, durante el evento construimos el dashboard y el simulador de tesorería, la integración web con el vault de Fuji, la API de registro y conciliación de préstamos y la automatización con ventana de veto. La aplicación usa [la ABI del vault](web/src/lib/vault-abi.ts). La colocación de excedentes y la integración con Belo se presentan como simulaciones.
