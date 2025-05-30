# Variable Naming for Code Reviews

## Core Philosophy

**ALL variables MUST be highly descriptive to ensure code maintainability and clarity.**

## The Three-Question Test (MANDATORY)

Every variable name must answer these three questions clearly:

1. **What** is it exactly?
2. **Where** does it come from or what generates it?
3. **Why** does it exist or what is its purpose?

## Critical Rules

### Zero Ambiguity Rule

If any developer needs to think "what does this mean?" when reading a variable name, the name has failed and must be changed.

### The Six-Month Test

Variable names must pass this test: _Would someone who has never seen this code understand what this variable contains just from its name after 6 months?_

### Accuracy Rule

Variable names must accurately reflect their actual content/source, not assumptions.

- ❌ BAD: `currentIframeViewportWidth` for `window.innerWidth`
- ✅ GOOD: `currentBrowserWindowInnerWidth`

## BANNED VARIABLE NAMES (Automatic Rejection)

### Generic Names

- `data`, `item`, `temp`, `flag`, `val`, `result`, `obj`, `str`, `p`

### Ambiguous Booleans

- `isOk`, `isReady`, `isActive`, `isValid`, `isLeft`, `isRight`

### Vague Positions/Dimensions

- `left`, `top`, `width`, `height`, `x`, `y`

### Generic Loops

- `i`, `j`, `k` (use descriptive names instead)

### Unclear Timing

- `time`, `duration`, `delay`

### Generic Elements

- `element`, `container`, `div`, `span`

### Vague States

- `status`, `state`, `mode`

## REQUIRED STANDARDS

### Position/Dimension Variables

- ❌ BAD: `videoLeft`, `videoTop`, `left`, `right`
- ✅ GOOD: `videoElementLeftBoundaryPixels`, `badgeContainerLeftPositionRelativeToVideo`

### Boolean Variables

- ❌ BAD: `isOk`, `isReady`, `isActive`
- ✅ GOOD: `isVideoPlaybackCurrentlyActive`, `isDrmLicenseSuccessfullyValidated`, `isFullscreenModeCurrentlyEnabled`

### Timing/Duration Variables

- ❌ BAD: `time`, `duration`, `delay`
- ✅ GOOD: `playbackPositionInSeconds`, `totalVideoDurationInMinutes`, `badgeDisplayIntervalInMilliseconds`

### Element/DOM Variables

- ❌ BAD: `element`, `container`, `div`
- ✅ GOOD: `badgeContainerElement`, `videoPlayerControlsElement`, `protectionOverlayBackgroundElement`

### State/Status Variables

- ❌ BAD: `status`, `state`, `mode`
- ✅ GOOD: `currentProtectionLevelStatus`, `playbackSessionConnectionState`, `drmSystemInitializationMode`

### Loop Variables

- ❌ BAD: `i`, `j`, `temp`, `val`
- ✅ GOOD: `currentTrackIndex`, `protectionLevelIterationIndex`, `temporaryEncryptedLicenseData`

## Variable Categories

### Instance Variables

- camelCase for public properties
- Examples: `shakaPlayerInstance`, `videoPlayerElement`, `drmSystemsManagerServiceInstance`

### Private Variables

- Underscore prefix with camelCase
- Examples: `_encryptionPrivateKey`, `_badgeContainerElement`, `_currentProtectionLevelStatus`

### Configuration Objects

- camelCase with "Configuration" suffix
- Examples: `colorsConfiguration`, `bigPlayButtonConfiguration`, `drmSystemsConfiguration`

### Constants

- SCREAMING_SNAKE_CASE for static values
- Examples: `ELEMENT_ID_FOR_BADGE_CONTAINER`, `Z_INDEX_FOR_PLAYER_UI`

### Bound Event Handlers

- Use `_bound` prefix for bound event handlers
- Examples: `_boundHandleKeyDown`, `_boundHandleResize`, `_boundHandleVideoPlaybackStateChange`

## Code Review Enforcement

### Immediate Rejection Triggers

1. **Any banned variable name** from the list above
2. **Single-letter variables** (except specific mathematical contexts)
3. **Generic parameter names**: `obj`, `str`, `p`, `val`, `result`
4. **Variables that fail the Three-Question Test**
5. **Abbreviations without context**: `cfg`, `mgr`, `btn`, `elem`

### Required Verification Checklist

Before approving any code, verify:

- [ ] Every variable name clearly states WHAT it contains
- [ ] Every variable name indicates WHERE the data comes from
- [ ] Every variable name explains WHY it exists
- [ ] Variable names accurately match their actual data source
- [ ] No mental translation is required to understand the variable's purpose
- [ ] Boolean variables describe the exact condition being checked
- [ ] Position/dimension variables specify units and reference points
- [ ] Timing variables include time units
- [ ] Element variables specify their DOM relationship and purpose

### Code Review Questions

1. "Can I understand what this variable contains without looking at its assignment?"
2. "Does this variable name accurately reflect where the data comes from?"
3. "Would this variable name still make sense in 6 months?"
4. "Is there any ambiguity about what this variable represents?"

### Enforcement Standards

- **Zero tolerance** for banned variable names
- **Require explicit justification** for any variable name shorter than 15 characters
- **Mandate rewrites** for any code that fails the Six-Month Test
- **Automatic approval block** for code containing generic names

## Remember

**Clarity is not optional. Self-documenting code is the only acceptable code.**
