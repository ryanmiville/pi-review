# Pull Request Forge Abstraction Spec

Target file: `review.ts`

Goal: support any code forge with a PR-like flow using one internal pull request abstraction.

Canonical command:

```txt
/review pr <number-or-url>
```

Examples:

```txt
/review pr 123
/review pr https://github.com/owner/repo/pull/123
/review pr https://gitlab.com/group/project/-/merge_requests/123
```

Do not keep `/review mr` unless it clearly improves maintainability/simplicity. Current decision: remove it.

## Style constraints

Follow existing `review.ts` style.

Do:

- use plain `string` and `number`
- use existing discriminated-union style
- use `Promise<T | null>` and `{ success: boolean; error?: string }`
- keep helpers as functions/objects, not classes
- keep UI notifications at resolver/command boundary

Do not:

- add branded/newtype aliases
- add provider id unions like `"github" | "gitlab"`
- add Result monads
- leak concrete forge implementation types into `ReviewTarget`

## File organization

Default: keep forge implementations in `review.ts`.

Reason: package is currently single-file and GitHub/GitLab adapters are small. Splitting early adds module surface without clear payoff.

Add a comment near the forge code:

```ts
// Pull request forge support
// Keep forge implementations here while there are only a few small adapters.
// If adapters grow substantial or a third forge is added, extract to pull-request-forges.ts.
```

Recommended in-file order:

1. `PullRequestInfo`
2. `PullRequestForge`
3. generic helpers
4. GitHub forge object
5. GitLab forge object
6. `PULL_REQUEST_FORGES`
7. resolver/UI functions

Extract only if earned by one of:

- third forge added
- forge code exceeds ~150-200 lines
- tests need importing pure parsing helpers
- provider behavior becomes complex enough to obscure review command flow

If extracted later, use:

```txt
pull-request-forges.ts
review.ts
```

`pull-request-forges.ts` owns:

- `PullRequestInfo`
- `PullRequestForge`
- `PULL_REQUEST_FORGES`
- forge URL parsers
- `parsePullRequestNumber`
- `findForgeForUrlReference`
- `getRemoteUrls`
- `inferPullRequestForge`

`review.ts` owns:

- UI notifications
- forge selector
- `ensurePullRequestForgeReady`
- `resolvePullRequestTarget`
- command parsing
- prompt generation

If extracted, import with `.ts` extension because current `tsconfig.json` uses `NodeNext` + `allowImportingTsExtensions`:

```ts
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { PULL_REQUEST_FORGES } from "./pull-request-forges.ts";
```

Avoid circular imports.

## Target types

Replace separate PR/MR target variants with one pull request variant:

```ts
type ReviewTarget =
	| { type: "uncommitted" }
	| { type: "baseBranch"; branch: string }
	| { type: "commit"; sha: string; title?: string }
	| {
			type: "pullRequest";
			number: number;
			baseBranch: string;
			title: string;
			shortName: string; // "PR", "MR", etc.
			refPrefix: string; // "#", "!", etc.
		}
	| { type: "folder"; paths: string[] };
```

Shared metadata:

```ts
type PullRequestInfo = {
	baseBranch: string;
	title: string;
	headBranch: string;
};
```

Forge seam:

```ts
type PullRequestForge = {
	name: string; // "GitHub", "GitLab"
	cliName: string; // "gh", "glab"
	setupInstructions: string;
	authStatusArgs: string[];
	authLoginCommand: string;
	shortName: string; // "PR", "MR"
	refPrefix: string; // "#", "!"

	parseUrlReference(ref: string): number | null;
	matchesRemote(remoteUrl: string): boolean;
	getInfo(pi: ExtensionAPI, number: number): Promise<PullRequestInfo | null>;
	checkout(pi: ExtensionAPI, number: number): Promise<{ success: boolean; error?: string }>;
};
```

Registry:

```ts
const PULL_REQUEST_FORGES: PullRequestForge[] = [
	GITHUB_PULL_REQUEST_FORGE,
	GITLAB_PULL_REQUEST_FORGE,
];
```

## Parser rule

Do not let forge adapters parse bare numbers.

Use one generic helper:

```ts
function parsePullRequestNumber(ref: string): number | null
```

Forge adapters only parse forge-specific URLs:

- GitHub: `github.com/owner/repo/pull/123`
- GitLab: `gitlab.com/group/project/-/merge_requests/123`

This prevents bare `123` from selecting whichever forge is first in the registry.

## Forge selection API

Add helpers:

```ts
function findForgeForUrlReference(
	ref: string,
): { forge: PullRequestForge; number: number } | null
```

```ts
async function getRemoteUrls(pi: ExtensionAPI): Promise<string[]>
```

```ts
async function inferPullRequestForge(pi: ExtensionAPI): Promise<PullRequestForge | null>
```

Inference order:

1. `git remote get-url origin`
2. fallback `git remote -v`
3. return forge if exactly/clearly matched
4. return `null` if unknown/ambiguous

For unknown bare numbers, ask user:

```ts
async function selectPullRequestForge(ctx: ExtensionContext): Promise<PullRequestForge | null>
```

Selector labels use `forge.name`.

## Generic resolver

Replace both current PR/MR resolvers with:

```ts
async function resolvePullRequestTarget(
	ctx: ExtensionContext,
	ref: string,
	options: { skipInitialPendingChangesCheck?: boolean } = {},
): Promise<ReviewTarget | null>
```

Flow:

1. check pending changes unless skipped
2. try URL parse via `findForgeForUrlReference(ref)`
3. if not URL:
   - parse bare positive number
   - infer forge from remotes
   - if inference fails, ask user to choose forge
4. ensure forge CLI ready with generic helper:

```ts
async function ensurePullRequestForgeReady(
	ctx: ExtensionContext,
	forge: PullRequestForge,
): Promise<boolean>
```

5. fetch info: `forge.getInfo(pi, number)`
6. re-check pending changes
7. checkout: `forge.checkout(pi, number)`
8. return unified target:

```ts
return {
	type: "pullRequest",
	number,
	baseBranch: info.baseBranch,
	title: info.title,
	shortName: forge.shortName,
	refPrefix: forge.refPrefix,
};
```

## Prompt/hint changes

Remove separate PR/MR prompt constants.

Use one prompt pair:

```ts
const PULL_REQUEST_PROMPT =
	'Review {shortName} {refPrefix}{number} ("{title}") against the base branch \'{baseBranch}\'. The merge base commit for this comparison is {mergeBaseSha}. Run `git diff {mergeBaseSha}` to inspect the changes that would be merged. Provide prioritized, actionable findings.';

const PULL_REQUEST_PROMPT_FALLBACK =
	'Review {shortName} {refPrefix}{number} ("{title}") against the base branch \'{baseBranch}\'. Start by finding the merge base between the current branch and {baseBranch} (e.g., `git merge-base HEAD {baseBranch}`), then run `git diff` against that SHA to see the changes that would be merged. Provide prioritized, actionable findings.';
```

`getUserFacingHint`:

```ts
return `${target.shortName} ${target.refPrefix}${target.number}: ${shortTitle}`;
```

## Command/UI changes

Header docs:

- remove `/review mr ...`
- document GitLab MR URL under `/review pr ...`

Preset list:

```ts
{ value: "pullRequest", label: "Review a pull request", description: "(GitHub/GitLab)" }
```

Parse args:

- keep `case "pr"`
- remove `case "mr"`

Parsed async sentinel:

```ts
type ParsedReviewArgs = {
	target: ReviewTarget | { type: "pr"; ref: string } | null;
	extraInstruction?: string;
	error?: string;
};
```

Input prompt:

```ts
"Enter PR number or URL (GitHub PR or GitLab MR):"
```

## Phases

### Phase 1: internal seam + unified target

Do:

- add `PullRequestInfo`, `PullRequestForge`
- add GitHub/GitLab forge objects
- add forge registry
- add generic resolver
- change `ReviewTarget` to one `pullRequest`
- unify prompt/hint
- update current PR/MR call sites to use generic resolver

May temporarily leave `/review mr` only if needed during refactor, but route it through same resolver.

Verify:

```sh
npm run typecheck
```

Handoff note must include:

- whether `/review mr` still exists
- any remaining old functions/constants
- typecheck result

### Phase 2: command/UI simplification

Do:

- remove `/review mr`
- remove MR selector entry
- remove old MR-only helpers/constants
- ensure `/review pr <gitlab-mr-url>` works through forge URL parsing
- ensure `/review pr 123` infers remote or prompts forge

Verify:

```sh
npm run typecheck
```

Handoff note must include:

- final command syntax
- remaining known limitations
- typecheck result

## Acceptance criteria

Final state:

- one `ReviewTarget` pull request variant
- one resolver for GitHub/GitLab/future forges
- no provider id union
- no `/review mr`
- GitHub and GitLab support implemented as `PullRequestForge` objects
- prompts/hints use `shortName` + `refPrefix`
- adding another forge means adding one object to `PULL_REQUEST_FORGES`, not changing core flow
- `npm run typecheck` passes
