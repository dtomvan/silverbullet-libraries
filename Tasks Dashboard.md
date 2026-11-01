---
tags: meta/library
name: Library/dtomvan/Tasks Dashboard
description: Widget that shows the amount of outstanding tasks
---

# Tasks Dashboard

Simple tasks dashboard, leveraging the new [stats widgets](https://github.com/silverbulletmd/silverbullet-libraries/blob/main/Library/silverbulletmd/Dashboard.md) from the Dashboard library.

Adds several views as commands to the editor:

- ${widgets.commandButton "Tasks: All"}
- ${widgets.commandButton "Tasks: Missed"}
- ${widgets.commandButton "Tasks: Today"}
- ${widgets.commandButton "Tasks: Tomorrow"}
- ${widgets.commandButton "Tasks: Future"}

And also the `widgets.tasksOverview()` widget. This one requires installing the Dashboard library also, but the widget will give you a convenient little button to do so.

## Demo

![](https://github.com/dtomvan/silverbullet-libraries/raw/refs/heads/main/2026-11-01%2020-04-07.mp4)

## Usage

In some convenient page (like your ${widgets.commandButton("Index", "Navigate: Home")}), add:

```markdown
${widgets.tasksOverview()}
```

## Customization

The views can be customized as sidebars, if so desired, as usual, with [`view.defaults`](https://docs.silverbullet.md/Library/Std/Config):

```lua
config.set("view.defaults", {
  ["my.tasks.today"] = {
    open = true,
    dock = "lhs",
  },
  ["my.tasks.all"] = {
    dock = "bhs",
  }
})
```

## Implementation

### Queries and templates necessary

```space-lua
templates = templates or {}
templates.tasksPerDate = template.new[==[
# [[Journal/${page}]]
${table.concat(query[[from t = tasks order by t.due select templates.taskItem(t)]])}
]==]

queries = queries or {}

function queries.missedTasks()
  return query[[
    from t = index.tasks()
    where not t.done and t.due != nil and t.due < date.today()
    order by t.due asc
    select templates.taskItem(t)
  ]]
end

function queries.tasksToday()
  return query[[
    from t = index.tasks()
    where not t.done and t.due != nil and (t.due == date.today() or t.date == date.today())
    order by t.pageLastModified
    select templates.taskItem(t)
  ]]
end

function queries.tasksTomorrow()
  return query [[
    from t = index.tasks()
    where not t.done and t.due != nil and (t.due == date.tomorrow() or t.date == date.tomorrow())
    order by t.pageLastModified
    select templates.taskItem(t)
  ]]
end

function queries.tasksFuture()
  return query [[
    from t = index.tasks()
    where not t.done and t.due != nil and t.due > date.tomorrow()
  ]]
end

function queries.tasksFutureGrouped()
  return query [[
    from t = queries.tasksFuture()
    group by t.due
    order by key
    select templates.tasksPerDate {page=key,tasks=group}
  ]]
end
```

### Install Dashboard function

```space-lua
function installDashboard()
  local ok = pcall(share.install, "https://github.com/silverbulletmd/silverbullet-libraries/blob/main/Library/silverbulletmd/Dashboard.md")
  if ok then
    editor.flashNotification("Installed Dashboard library successfully, reloading...")
    system.reboot()
  end
end
```

### The views, commands, and Dashboard widget

```space-lua
local categories = {
  {
    viewName = "my.tasks.missed",
    statName = "missed",
    title = "Missed tasks",
    command = "Tasks: Missed",
    tone = "warning",
    q = queries.missedTasks
    },
  {
    viewName = "my.tasks.today",
    title = "Today's tasks",
    command = "Tasks: Today",
    statName = "today",
    tone = "success",
    q = queries.tasksToday
    },
  {
    viewName = "my.tasks.tomorrow",
    statName = "tomorrow",
    title = "Tomorrow's tasks",
    command = "Tasks: Tomorrow",
    tone = "neutral",
    q = queries.tasksTomorrow
    },
  {
    viewName = "my.tasks.future",
    statName = "in the future",
    title = "Tasks in the future",
    command = "Tasks: Future",
    tone = "neutral",
    -- HACK: little wasteful
    qForCount = queries.tasksFuture,
    q = queries.tasksFutureGrouped,
    },
  {
    viewName = "my.tasks.all",
    title = "All tasks",
    command = "Tasks: All",
    statName = "other",
    tone = "neutral",
    q = function() return query [[ from index.tasks() where not done select templates.taskItem(_) ]] end,
    }
  }

for _, stat in ipairs(categories) do
  view.define {
    name = stat.viewName,
    title = stat.title,
    command = stat.command,
    dock = "modal",
    refreshOn = { "index" },
    supportedDocks = { "lhs", "rhs", "bhs", "modal" },
    content = stat.q,
  }
end

widgets = widgets or {}

function widgets.tasksOverview()
  if not widgets.stat then
    return widget.markdownBlock([[**Error:** `widgets.stat` is not available. Did you ${widgets.button("Install", installDashboard)} the Dashboard library?]])
  end

  local cells = {}

  for i, c in ipairs(categories) do
    local fn = c.qForCount or c.q
    local rows = #fn()
    -- HACK: last category will only appear if there's no other stats displayed yet
    if (rows > 0 and i ~= #categories) or (i == #categories and #cells == 0) then
      table.insert(cells, widgets.stat(c.statName, rows, {
        tone = c.tone,
        onClick = function ()
          view.open(c.viewName)
        end
      }))
    end
  end

  return widgets.grid(cells)
end
```

