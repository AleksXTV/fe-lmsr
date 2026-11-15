# fe-lmsr: LMSR prediction markets in Fe that round in the market's favour

**Pitch:** A logarithmic market scoring rule (LMSR) market maker for 2 to 16
outcomes, written in Fe. Fixed-point rounding can never make the market
insolvent. A 20,000-case differential test against a 60-digit reference checks
this, and the gas cost is within 4% of the best Solidity build of the same
algorithm on Solady math.

## The problem

LMSR (Hanson, 2003) is the standard automated market maker for prediction
markets. Its cost function is `C(q) = b * ln(sum_i exp(q_i / b))`. On-chain it
has to run on fixed-point `exp` and `ln`, and those are only accurate to a few
wei. If a contract charges the *computed* cost difference `C(q') - C(q)`, a
trader can pick trades where rounding works for them, buy and sell repeatedly,
and drain the subsidy. Most on-chain LMSR code ignores this or adds an ad hoc
fee. Solidity also forces a choice between copy-pasting the market for every
outcome count and using dynamic arrays plus runtime checks.

## Approach

This project treats rounding as part of the specification:

1. `ingots/lmsr` computes `C` stably as `b * (m + ln(sum exp(q_i/b - m)))`,
   so every `exp` argument is at most zero. It also has an explicit error bound
   `error_bound()` and brackets the true cost: `cost_lower(q) <= C(q) <= cost_upper(q)`.
2. Buyers pay `cost_upper(after) - cost_lower(before)`. Sellers receive
   `cost_lower(before) - cost_upper(after)`. So every trade pays the market
   **at least** the true LMSR cost difference. The sponsor funds the market with
   `cost_upper(0) >= b ln N`. Together with `C(q) >= max_i q_i`, this means the
   market can always pay every share of whichever outcome wins.
3. `ingots/market` is a full ETH-settled market: buy and sell with slippage
   limits, oracle resolution, redemption, and a surplus sweep for the sponsor.
   Every state-changing handler re-checks `balance >= liability` against the
   real contract balance and reverts with `Insolvent` if it ever fails. If that
   check is ever reached, there is a bug.
4. The bracket is checked in three independent ways:
   - **Differential test** (`tools/difftest.py`). The same Fe library is
     compiled to a native binary (`ingots/lmsr-cli`) and compared on 20,000
     random markets with a 60-digit `Decimal` reference. Inputs span N in
     {2,3,4,8,16}, b from 0.001 to 10^12, and up to 10^15 shares.
   - **Reference vectors**: 300 generated cases in `ingots/lmsr/src/vectors.fe`,
     run by `fe test` in CI with no Python.
   - **Property and stateful fuzz tests** using `std::evm::random`:
     - round trips never profit
     - splitting an order gives no discount
     - prices are monotone probabilities
     - cost covers the largest payout
     - 120-step random campaigns on a binary and a 4-outcome market, checking
       accounting, holdings and solvency after every step, then resolving and
       redeeming down to an empty contract

Everything in `ingots/`, `tools/`, `bench/` and `examples/` was written during
the contest (see Declaration).

### Results (reproducible, `fe 26.5.1`)

Differential test, `python3 tools/difftest.py --cases 20000 --seed 1`:

```
cases: 20000  (N in [2, 3, 4, 8, 16], b from 0.001 to 1e12, shares up to 1e15)
bracket cost_lower <= C_true <= cost_upper: 20000/20000 hold
worst |cost - C_true|: 8522527186685 wei; worst error / error_bound: 0.1533
worst |price - p_true|: 13.1 wei; worst |sum(prices) - 1|: 15 wei
```

The largest absolute error (8.5e12 wei, i.e. 8.5e-6 shares) comes from
markets with b near 10^12. Even there it uses only 15% of the bound.

Gas, `fe bench bench --solc solc-0.8.28` (total call gas incl. the 21k base;
"best" is the cheapest of four solc builds: legacy, legacy+opt, IR, IR+opt):

| call | Fe (Sonatina) | solc legacy+opt | solc IR+opt | Fe vs best |
|---|---:|---:|---:|---:|
| quoteBuy, 2 outcomes, empty | 28,489 | 29,845 | 28,696 | -0.7% |
| quoteBuy, 2 outcomes, skewed | 28,670 | 30,021 | 28,884 | -0.7% |
| quoteBuy, 4 outcomes | 34,057 | 35,262 | 33,562 | +1.5% |
| price, 4 outcomes | 28,007 | 27,451 | 26,953 | +3.9% |

`bench/lmsr.sol` is a hand port of the same algorithm, with the same 512-bit
mul/div and the same error margins, onto Solady's `FixedPointMathLib`. Fe's
`core::fixed` exp/ln are themselves Solady ports. `bench/lmsr.fe` is generated
from the library by `tools/make_bench.py`, so the benchmarked code is the
shipped code. The bench harness measures gas only. It does not compare return
values between the Fe and Solidity versions.

## Fe highlight

**The outcome count is a type parameter with a compile-time bound, and storage
reaches the generic core through effects.** One generic core serves every
market size, and an invalid size fails to compile:

```fe
pub struct Lmsr<const N: usize> where N >= 2, N <= 16 { pub b: Wad }

pub struct Book<const N: usize> where N >= 2, N <= 16 {
    pub b: Wad, pub q: [Wad; N], pub oracle: Address, pub sponsor: Address,
    pub resolved: bool, pub winner: u8, pub owed: Wad,
}

pub fn buy<const N: usize>(outcome: u8, shares: Wad, max_cost: Wad) -> Wad
uses (book: mut Book<N>, holdings: mut Holdings, ctx: Ctx, call: mut Call, log: mut Log)
where N >= 2, N <= 16
{ ... }

pub contract CategoricalMarket4 uses (ctx: Ctx, call: mut Call, log: mut Log) {
    mut book: Book<4>,
    mut holdings: Holdings,
    recv MarketMsg {
        #[payable]
        Buy { outcome, shares, max_cost } -> u256
        uses (mut book, mut holdings, ctx, mut call, mut log) {
            with (book, holdings) {
                buy<4>(outcome: outcome, shares: Wad::from_raw(shares), max_cost: Wad::from_raw(max_cost)).raw()
            }
        }
        ...
```

The market logic never names a storage slot. It declares the capabilities it
needs (`uses`), and each contract provides its own storage with `with (...)`.
Adding a 3- or 8-outcome market is one short contract. `examples/compile-fail`
tries `Lmsr<1>`, and the compiler rejects it (checked in CI):

```
error[8-0093]: const requirement is not satisfied
 5 │     let m: Lmsr<1> = Lmsr<1>::new(b: Wad::from_raw(1000000000000000000))
   │            ^^^^^^^ condition evaluated to `false`
22 │ pub struct Lmsr<const N: usize> where N >= 2, N <= 16 { pub b: Wad }
   │                                       ------ required by this predicate
```

The same effect discipline makes the tests honest. `pay()` declares
`uses (call: mut Call, ctx: Ctx)`, so a reader can see which functions can make
external calls. A `Reenterer` contract in the tests sells during the refund of
its own buy and observes the updated state.

Other Fe features used: `core::fixed` (`Wad`/`SWad`, `exp`, `ln`),
`std::evm::random` (`KeccakRng`) for property tests, `#[event]`/`#[error]` with
`revert_error` and `is_error<E>` in tests, Solidity-compatible selectors, the
native backend (`fe build --backend native`, `std::io`) for the differential
test, and `fe bench` for gas.

## Architecture

```
fe.toml                     workspace: lmsr, market, lmsr-cli
ingots/lmsr/                pure LMSR math, no EVM dependencies
  src/lib.fe                Lmsr<N>: cost, cost_lower/upper, error_bound, funding, price, buy_cost, sell_return
  src/tests.fe              unit + KeccakRng property tests
  src/vectors.fe            300 generated reference cases (tools/difftest.py)
ingots/market/              ETH-settled markets on top of lmsr
  src/lib.fe                Book<N>, generic buy/sell/resolve/redeem/sweep, events, errors,
                            BinaryMarket (Book<2>), CategoricalMarket4 (Book<4>)
  src/tests.fe              lifecycle, slippage, funding, authorization, reentrancy (Trader/Reenterer proxies)
  src/invariants.fe         stateful fuzz campaigns with solvency checks
ingots/lmsr-cli/            the same lmsr ingot compiled natively: reads "N b q..." lines,
                            prints cost, lower, upper, prices
tools/difftest.py           20k-case differential test vs 60-digit Decimal; regenerates vectors.fe
tools/make_bench.py         generates bench/lmsr.fe from ingots/lmsr/src/lib.fe
bench/                      fe bench fixture: lmsr.fe, lmsr.sol (Solady), lmsr.toml, results/
examples/compile-fail/      Lmsr<1>, must not compile
```

Contract interface (`BinaryMarket` / `CategoricalMarket4`):

- `constructor(uint256 b, address oracle)`, payable, must send at least
  `funding = cost_upper(0)`, about `b ln N`
- `buy(uint8 outcome, uint256 shares, uint256 maxCost) payable returns (uint256 cost)`.
  Change is refunded.
- `sell(uint8 outcome, uint256 shares, uint256 minReturn) returns (uint256 payout)`
- `resolve(uint8 outcome)`, oracle only
- `redeem() returns (uint256)`: 1 ether per winning share
- `sweep(address to) returns (uint256)`: sponsor only, after resolution, sends
  everything that is not owed to winners
- views: `price(uint8)`, `quoteBuy(uint8,uint256)`, `quoteSell(uint8,uint256)`,
  `sharesOf(address,uint8)`, `outstanding(uint8)`, `liquidity()`, `outcomes()`
- events `Bought`, `Sold`, `Resolved`, `Redeemed`, plus custom errors
  (`SlippageExceeded`, `InsufficientPayment`, `Underfunded`, `Insolvent`, ...)

Runtime bytecode: `BinaryMarket` 4,076 bytes, `CategoricalMarket4` 4,151 bytes.

## Build and test

```sh
fe --version                 # fe 26.5.1 (149c758)
fe test                      # 30 tests: unit, property, 300 reference vectors, market, fuzz
fe build                     # EVM bytecode + ABI in ./out
fe check examples/compile-fail   # expected to FAIL: Lmsr<1> violates N >= 2

# differential test (Python 3 standard library only)
fe build --backend native --out-dir ingots/lmsr-cli/out ingots/lmsr-cli
python3 tools/difftest.py --cases 20000 --vectors 0

# gas vs Solidity (needs solc, e.g. the static 0.8.28 binary from the Solidity releases)
fe bench bench --solc /path/to/solc
```

CI (`.github/workflows/ci.yml`) runs all of the above with `fmt --check`.

## Demo

Reproducible CLI example. The native quoter prints `cost lower upper price_0 ... price_{N-1}`
in wei:

```sh
$ printf '2 100000000000000000000 0 0\n2 100000000000000000000 60000000000000000000 0\n' | ingots/lmsr-cli/out/lmsr_cli
69314718055994530900 69314718055994526796 69314718055994535004 500000000000000000 500000000000000000
103748795048588562600 103748795048588558496 103748795048588566704 645656306225795453 354343693774204546
```

Line 1 is an empty binary market with b = 100. The true `C(0) = 100 ln 2 =
69.314718055994530941...`, so the computed value is 41 wei off. The bracket is
±4,104 wei, and that bracket width is all the market pays for its safety. Line 2 is
the same market after 60 YES shares, where YES is priced at `e^0.6/(1+e^0.6) = 0.64566`.

The test `full_lifecycle_pays_winners_and_returns_surplus` is a walk-through of a whole
market: a sponsor funds it, two traders buy opposite sides, the oracle resolves,
the winner redeems, and the sponsor sweeps the rest to a treasury, leaving the contract at zero.

## Limitations

- **Trusted oracle.** One address resolves the market. There is no dispute
  window, no deadline, and no invalid/cancel outcome.
- **No fees.** The market maker's worst-case loss is the subsidy `b ln N`
  (plus the bracket margins, which work in its favour).
- **Shares are not transferable tokens.** Holdings are an internal map with no
  ERC-1155 interface.
- **ETH only.** No ERC-20 collateral.
- **Bounds:** b in [0.001, 10^12], at most 10^15 shares per outcome, N in [2, 16].
  `error_bound()` is an analytic bound with a wide margin, chosen and then
  validated empirically (worst observed use is 15%). It is not a machine-checked proof.
- **Tooling quirks:** `fe build` warns that the test-only `Trader` proxy has
  recv arms without `sol` selectors. That is intentional, and the warning does
  not affect the market contracts.
- Not audited and not deployed to mainnet. Use at your own risk.

## Declaration

- **Pre-existing work:** None of the project code existed before November 3, 2026.
  The repository was started from the official
  [builders-challenge-starter](https://github.com/fe-lang/builders-challenge-starter)
  at commit `f9d9aa9`. From it, this project keeps only `LICENSE`, the shape of the CI
  workflow, and the structure of this README (the template sections). The
  starter's example code was removed.
- **Third-party code:**
  - `bench/lmsr.sol` embeds Solady's `FixedPointMathLib`
    (MIT, https://github.com/Vectorized/solady) and is used for gas comparison only.
  - Fe's `core::fixed` ships with the Fe compiler.
- **AI assistance:** AI coding assistants helped write and review the Fe code,
  the tests, the Python tooling and this README. All results quoted here were
  produced by running the commands above.

## Team

Aleksandr Kulikov (solo).

## License

Apache-2.0 (see `LICENSE`). `bench/lmsr.sol` contains MIT-licensed Solady code.
