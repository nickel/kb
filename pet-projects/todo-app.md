# To-Do APP

I want to create a TUI app to handle a To-Do app.
Append-only event log. Every action is an event (added, completed, snoozed, retitled), state is a fold over the log. Sync across machines becomes log merge instead of state merge, you get an undo stack and a real audit trail of your day, and last-write-wins per field handles the rare conflict. More fun to build, and it plays to what you already think about.
I don't want to use the mouse, the way to interact should be only the keyboard.
Modal vim bindings: h/j/k/l, o to insert below, dd, u, / to filter, : for a command line. Fuzzy filter that narrows as you type beats any menu.
A "today" view that's a queue, not a list — one task at the top, space to complete, s to snooze until tomorrow/next week. The full list is a separate screen you visit rarely.
Instant capture as its own tiny mode: launch, type, enter, exit. Bind it to a floating terminal hotkey so it's a half-second interruption. This is the feature you'll actually use daily, so optimize its startup path.
Undo as a first-class key, not a menu item. Makes destructive keys safe, which makes the whole thing feel fast.
The task can be part of another task (moving a task to the right or creating a subtask for a task).
If a mark a parent task as completed, subtasks will be marked as well, and moved to another section (completed).
I can uncomplete tasks, but subtasks only can be uncompleted when the parent task is uncompleted.
The keys should be configurable on ~/.config/<app_name> folder.
Timers. t starts a timer on the selected task, stopping it writes a duration event. After a week you have real data on where time went.
