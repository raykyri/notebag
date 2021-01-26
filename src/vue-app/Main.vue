<template>
	<div class="main layout flex">
		<viewbar />
		<div ref="layout" class="nv-layout" v-show="mode === Modes.NOTES">
			<note-overview />
			<editor />
                </div>
		<project-overview v-show="mode === Modes.CATEGORIES" />
		<note-overview v-show="mode === Modes.ARCHIVE" />
	</div>
</template>

<script>
/* eslint-disable indent */
	import { ipcRenderer } from "electron";
	import { mapState } from "vuex";
	import Mousetrap from "mousetrap";
	import throttle from "lodash.throttle";

	import EventBus, {Events} from "@/EventBus";

	import electronStore from "@/store/electronStore";
	import Modes from "@/store/modes";
	import Actions from "@/store/actions";
	import Preferences from "@/store/preferences";
	import Payloads from "@/store/payloads";
	import Shortcuts, { defaultShortcuts, shortcutOs } from "@/store/shortcuts";

	import ProjectOverview from "@/components/ProjectOverview";
	import NoteOverview from "@/components/NoteOverview";

	import Viewbar from "@/components/Viewbar";
	import Editor from "@/components/Editor";

	import { makeAccelerator, redefineMousetrapShortcut } from "@/utilities/shortcuts";


	export default {
		name: "Main",
		components: {
			ProjectOverview,
			NoteOverview,
			Viewbar,
			Editor,
		},
		computed: {
			...mapState([ "mode", "activeNote", "starredNotes" ]),
			theme() {
				return this.$store.state.preferences.theme;
			},
			pinnedProjects() {
				return Object.values(this.$store.state.projects).filter(project => project.pinned);
			}
		},
		methods: {
			[Shortcuts.GO_UP_NOTE]: throttle(function() {
				EventBus.$emit(Events.REQUESTING_PREV_NOTE);
				document.querySelector(".note-overview__search").blur();
			}, 50),
			[Shortcuts.GO_DOWN_NOTE]: throttle(function() {
				EventBus.$emit(Events.REQUESTING_NEXT_NOTE);
				document.querySelector(".note-overview__search").blur();
			}, 50),
			[Shortcuts.FOCUS_SEARCH]() {
				document.querySelector(".note-overview__search").focus();
				document.querySelector(".note-overview__search").select();
			},
			[Shortcuts.NEW_NOTE]() {
				document.querySelector(".note-overview__search").focus();
				document.querySelector(".note-overview__search").select();
			},
			[Shortcuts.DELETE_NOTE]() {
				if (!this.activeNote) {
					return;
				}

				this.$store.dispatch(Actions.DELETE_NOTE, this.activeNote);
			},
			[Shortcuts.TOGGLE_THEME]() {
				const newTheme = this.theme === Payloads.THEME_LIGHT
					? Payloads.THEME_DARK
					: Payloads.THEME_LIGHT;

				this.$store.commit({
					type: Actions.SET_PREFERENCE,
					preference: Preferences.THEME,
					value: newTheme,
				});
			},
			setMode(mode) {
				this.$store.commit(Actions.SET_MODE, mode);
			},
			setActiveProject(projectIndex) {
				const project = this.pinnedProjects[projectIndex];
				if (project) {
					this.$store.commit(Actions.SET_ACTIVE_PROJECT, project.tag);
					this.$store.commit(Actions.SET_MODE, Modes.CATEGORIES);
				}
			},
			migrateNotes(links, notes) {
				this.$store.commit(Actions.SET_LINKS, links);
				this.$store.commit(Actions.SET_NOTES, notes);
			},
			addImportedNote(_, note) {
				note = JSON.parse(note);
				this.$store.dispatch({
					type: Actions.ADD_NOTE,
					switchToNote: "no",
					...note
				});
			}
		},
		data() {
			return {
				Modes,
			};
		},
		created() {
			const redefineShortcut = redefineMousetrapShortcut(this);

			const newNoteAccelerator = makeAccelerator(
				electronStore.get(`preferences.shortcuts.${Shortcuts.NEW_NOTE}`, defaultShortcuts[shortcutOs][Shortcuts.NEW_NOTE]),
				true
			);
			const deleteNoteAccelerator = makeAccelerator(
				electronStore.get(`preferences.shortcuts.${Shortcuts.DELETE_NOTE}`, defaultShortcuts[shortcutOs][Shortcuts.DELETE_NOTE]),
				true
			);

			const toggleThemeAccelerator = makeAccelerator(
				electronStore.get(`preferences.shortcuts.${Shortcuts.TOGGLE_THEME}`, defaultShortcuts[shortcutOs][Shortcuts.TOGGLE_THEME]),
				true
			);

			const goUpOneNoteAccelerator = makeAccelerator(
				electronStore.get(`preferences.shortcuts.${Shortcuts.GO_UP_NOTE}`, defaultShortcuts[shortcutOs][Shortcuts.GO_UP_NOTE]),
				true
			);

			const goDownOneNoteAccelerator = makeAccelerator(
				electronStore.get(`preferences.shortcuts.${Shortcuts.GO_DOWN_NOTE}`, defaultShortcuts[shortcutOs][Shortcuts.GO_DOWN_NOTE]),
				true
			);

			const focusSearchBarAccelerator = makeAccelerator(
				electronStore.get(`preferences.shortcuts.${Shortcuts.FOCUS_SEARCH}`, defaultShortcuts[shortcutOs][Shortcuts.FOCUS_SEARCH]),
				true
			);

			Mousetrap.prototype.stopCallback = () => false;
			Mousetrap.bind(newNoteAccelerator, this[Shortcuts.NEW_NOTE]);
			Mousetrap.bind(deleteNoteAccelerator, this[Shortcuts.DELETE_NOTE]);
			Mousetrap.bind(toggleThemeAccelerator, this[Shortcuts.TOGGLE_THEME]);

			Mousetrap.bind(goUpOneNoteAccelerator, this[Shortcuts.GO_UP_NOTE]);
			Mousetrap.bind(goDownOneNoteAccelerator, this[Shortcuts.GO_DOWN_NOTE]);
			Mousetrap.bind(focusSearchBarAccelerator, this[Shortcuts.FOCUS_SEARCH]);

			const modifier = makeAccelerator(defaultShortcuts[shortcutOs][Shortcuts.TAB_SWITCH_MODIFIER], true);
			Mousetrap.bind(`${modifier}+1`, () => this.setMode(Modes.NOTES));
			Mousetrap.bind(`${modifier}+2`, () => this.setMode(Modes.CATEGORIES));
			Mousetrap.bind(`${modifier}+3`, () => this.setMode(Modes.ARCHIVE));

			for (let i=4; i<=9; i++) {
				Mousetrap.bind(`${modifier}+${i}`, () => this.setActiveProject(i-4));
			}

			ipcRenderer.on(`shortcut-changed:${Shortcuts.DELETE_NOTE}`, (_, oldKeys, newKeys) => redefineShortcut(Shortcuts.DELETE_NOTE, oldKeys, newKeys));
			ipcRenderer.on(`shortcut-changed:${Shortcuts.TOGGLE_THEME}`, (_, oldKeys, newKeys) => redefineShortcut(Shortcuts.TOGGLE_THEME, oldKeys, newKeys));

			ipcRenderer.on(`shortcut-changed:${Shortcuts.GO_UP_NOTE}`, (_, oldKeys, newKeys) => redefineShortcut(Shortcuts.GO_UP_NOTE, oldKeys, newKeys));
			ipcRenderer.on(`shortcut-changed:${Shortcuts.GO_DOWN_NOTE}`, (_, oldKeys, newKeys) => redefineShortcut(Shortcuts.GO_DOWN_NOTE, oldKeys, newKeys));
			ipcRenderer.on(`shortcut-changed:${Shortcuts.FOCUS_SEARCH}`, (_, oldKeys, newKeys) => redefineShortcut(Shortcuts.FOCUS_SEARCH, oldKeys, newKeys));

			ipcRenderer.on("recreate-tutorial", () => this.$store.commit(Actions.ADD_TUTORIAL_NOTE));
			ipcRenderer.on("imported-note", this.addImportedNote);

			EventBus.$on(Events.MIGRATED_NOTES, this.migrateNotes);
		}
	};
</script>

<style lang="scss">
	.nv-layout {
		width: calc(100vw - 52px);
	}
</style>
