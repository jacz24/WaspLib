# Simba-AI fork of WaspLib: the ledger

Branch `simba-ai`, cut from upstream `release` at `2026.07.31-e80c66a`
(commit `8833c68`). Every change to this checkout has an entry here: what,
why, the Simba-AI issue, and the test or probe that pins it. The KB pointer
rows live in `simba-kb/wasplib/fork-delta.md` in the Simba-AI repo.
`git diff upstream/release` is the live delta; this file is the reason for
each hunk. An upstream PR is a choice after our own testing, never a
requirement (Simba-AI charter amendment A10; #193, #204, #205).

| # | Date | Files | What | Why | Issue | Pinned by |
|---|---|---|---|---|---|---|
| 1 | 2026-09-18 | `main/position/map/objects.simba` (`TObject.Hover`), `main/position/map/entities.simba` (`TEntity.Hover`) | The body after the visibility check runs inside `try` / `finally`, and the `finally` clears `Walker^.TargetUpText` and `Walker^.ActionUpText`, as `WalkHover` already does on its own path | `_UpTextCheck` writes the object's name into the walker's `TargetUpText`, and the non-walking path never cleared it. Every later screen-walk click then treated that name as its target: after `doorObj.Interact(['Open'])` the next leg hovered a tile whose uptext read `Close Door`, opened the menu, and selected `Close Door` instead of `Walk here` (Simba-AI session `2026-09-18T13-23-11`, both attempts, the door closed behind the player). Operator ruling 2026-09-18: fix in the fork so every caller is covered, not in each caller | Simba-AI #229 | `Include/Travel/tests/test_walker_uptext_cleared.simba` (headless, image target, RED before / GREEN after); runtime confirmation on the next slice run through the `walker_target_uptext` field of Travel's `travel_leg` event |
