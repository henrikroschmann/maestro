# Maestro - Foundry VTT Module Development Guide

Maestro is a **Foundry Virtual Tabletop module** that adds audio features like Hype Tracks, Item Tracks, and Combat Tracks. Understanding Foundry VTT's architecture and hooks system is essential for development.

## Architecture Overview

### Conductor Pattern (maestro.js)
- **Main orchestrator** that registers all Foundry VTT hooks and coordinates module lifecycle
- Uses static methods to organize hook registration in `_initHookRegistrations()` and `_readyHookRegistrations()`
- Exposes public API via `game.maestro.*` namespace (e.g., `game.maestro.playHype`, `game.maestro.pause`)

### Core Track Classes
- **HypeTrack**: Actor-based audio that plays during combat turns
- **ItemTrack**: Audio triggered when items are rolled/used  
- **CombatTrack**: Playlist/track for entire combat encounters
- Each class follows pattern: constructor → static hook handlers → private methods → FormApplication

### Module Structure
```
modules/
├── config.js          # Constants, settings keys, default configurations
├── settings.js        # Foundry settings registration with onChange handlers
├── playback.js        # Audio playback utilities (pause, resume, play by name)
├── misc.js            # Critical success/failure sounds, playlist utilities
└── {track-type}.js    # Individual track implementations
```

## Foundry VTT Integration Patterns

### Hook Registration Pattern
```javascript
// Always use static methods in Conductor class
static _hookOnRenderActorSheet() {
    Hooks.on("renderActorSheet", (app, html, data) => {
        HypeTrack._onRenderActorSheet(app, html, data);
    });
}
```

### Flag-based Data Storage
Maestro stores configuration in **Foundry document flags**:
```javascript
// Actor hype track: actor.flags.maestro.track, actor.flags.maestro.playlist
// Combat track: combat.flags.maestro.combatStarted
// Always use config.js constants: MAESTRO.MODULE_NAME, flagNames
```

### Settings with onChange Handlers
```javascript
game.settings.register(MAESTRO.MODULE_NAME, key, {
    onChange: async s => {
        // Re-initialize playlists when settings change
        await game.maestro.hypeTrack._checkForHypeTracksPlaylist();
    }
});
```

## Development Workflows

### Build Process
- **No build step required** - pure ES6 modules loaded via `esmodules` in module.json
- Use `gulp docs` to generate API.md from JSDoc comments
- Version bumping requires updating both `package.json` and `module.json`

### Foundry VTT Version Support
- Target compatibility defined in `module.json` (currently v13)
- Use `@league-of-foundry-developers/foundry-vtt-types` for TypeScript definitions
- Check `game.version` for version-specific features

### Template System
HTML templates in `/templates/` use **Handlebars** syntax:
```html
<!-- Always include playlist selection pattern -->
<select class="playlist-select" name="playlist">
    {{#each playlists as |playlist|}}
    <option value="{{playlist.id}}">{{playlist.name}}</option>
    {{/each}}
</select>
```

## Key Conventions

### GM-Only Operations
```javascript
// Always check for first GM to avoid duplicate operations
import { isFirstGM } from "./misc.js";
if (!isFirstGM()) return;
```

### Playlist Management
- **Auto-creation**: Modules create playlists with standard names ("Hype Tracks", "Item Tracks")
- **Playback modes**: Support single track, random track, or full playlist via config constants
- **Pause/Resume**: Use `Playbook.pauseAll()` and `Playbook.resumeSounds()` for managing conflicts

### Internationalization
- All user-facing strings use `game.i18n.localize("MAESTRO.KEY")`
- Translation keys follow hierarchical pattern: `"MAESTRO.FEATURE.SubFeature.SpecificKey"`
- Default language is English (`lang/en.json`)

### Error Handling & Migration
- Migration system handles version upgrades via `modules/migration.js`
- Use Foundry's notification system: `ui.notifications.warn()`, `ui.notifications.info()`
- Store migration version in settings to prevent re-running

## External Dependencies
- **Foundry VTT API**: Core dependency - this is a Foundry module, not a standalone app
- **jQuery**: Available globally in Foundry for DOM manipulation
- **No build tools**: Pure ES6 modules, no bundling or transpilation needed

## Testing & Debugging
- Test in Foundry VTT world with module active
- Use browser dev tools - no special testing framework
- Check `game.maestro` object in console for API debugging
- Enable module settings to test different feature combinations