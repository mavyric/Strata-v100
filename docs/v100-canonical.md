# V100 (hooserv) canonical deployment

Canonical state as of 2026-10-04: rebased tree (upstream 99f3dbd) + Stage A
(F32 routers, iq_pack --compat-bf16) + the Strata#606 q8_1 activation-sum
fix.  This is what `custom-llamaswap:strata` serves on hooserv:8011.

- Pack: `q8_0_f32r` (F32 routers, BF16 GR rows).  Do NOT repack with
  `--native-gr`: the native Q8_0 GR mixer (6bdbaf9) breaks end-to-end
  coherence on the V100 - the model hallucinates truncated/obfuscated
  prompts at temp 0 - while gr_parity passes (it checks projections in
  isolation).  Bisect: pre-f32r coherent, f32r coherent, f32r+native-GR
  pack incoherent; the same binary with the old pack is coherent, so the
  defect is the native GR runtime/pack, not the rebase and not #606.
- The native-GR runtime code stays in the tree but is dormant: it activates
  only when the pack carries native GR index rows.
- Serve config carries `sampling: {temperature: 1.0, top_p: 0.95}` as the
  engine default (request fields still win; explicit temperature=0 = greedy).
- Debugging the native GR path (shared q8_1 scratch under graph replay is
  the prime suspect) is open work; until it passes a greedy probe battery
  through the live endpoint, keep the f32r pack.
