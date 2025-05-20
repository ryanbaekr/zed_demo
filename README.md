# Zed Demo

Zed is a relatively new editor that largely mimics the behavior and functionality of VS Code, but does so as a native application written in Rust. Ultimately, this allows Zed to be faster and more lightweight than VS Code.

The Zed GUI should feel familiar to VS Code users with project, git, and search panels available in the left dock and an editor pane with tabs sitting front and center.

![Image 1.](/assets/1.png "Image 1.")

Like VS Code, the editor pane can be split horizontally and vertically, panels can be moved, and a terminal panel can be opened at the bottom of the window to run commands right from the editor.

Zed supports extensions and themes, including several of each out of the box.

![Image 2.](/assets/2.png "Image 2.")

When it comes to AI integration, Zed's Inline Assist and Agent Panel can connect to LLMs running locally through Ollama.

![Image 3.](/assets/3.png "Image 3.")

![Image 4.](/assets/4.png "Image 4.")

Since Zed is open source, it can be modified to add functionality. It ended up being quite easy to add entries to the app menu and command palette.

![Image 5.](/assets/5.png "Image 5.")

![Image 6.](/assets/6.png "Image 6.")

With a little more work one can add a whole new page to Zed.

![Image 7.](/assets/7.png "Image 7.")

Here's some example code for a new page that includes embedded Python via PyO3.

``` rust
use editor::Editor;
use gpui::{
    App, Context, Entity, EventEmitter, Focusable, ParentElement, Render, Styled, Window, actions,
};
use paths::config_dir;
use std::{borrow::Borrow, fs};
use ui::prelude::*;
use workspace::{Item, Workspace, WorkspaceId, item::ItemEvent, notifications::NotifyResultExt};

use pyo3::prelude::*;
use pyo3_ffi::c_str;

actions!(sped, [OpenFolder]);
actions!(sped_page, [ToggleFocus]);

pub fn init(cx: &mut App) {
    cx.observe_new(register).detach();
}

fn register(workspace: &mut Workspace, _window: Option<&mut Window>, _: &mut Context<Workspace>) {
    workspace.register_action(open_folder);
    workspace.register_action(toggle_focus);
}

fn open_folder(
    workspace: &mut Workspace,
    _: &OpenFolder,
    _: &mut Window,
    cx: &mut Context<Workspace>,
) {
    fs::create_dir_all(config_dir().join("sped")).notify_err(workspace, cx);
    cx.open_with_system(config_dir().join("sped").borrow());
}

fn toggle_focus(
    workspace: &mut Workspace,
    _: &ToggleFocus,
    window: &mut Window,
    cx: &mut Context<Workspace>,
) {
    let existing = workspace
        .active_pane()
        .read(cx)
        .items()
        .find_map(|item| item.downcast::<SpedPage>());

    if let Some(existing) = existing {
        workspace.activate_item(&existing, true, true, window, cx);
    } else {
        let sped_page = SpedPage::new(window, cx);
        workspace.add_item_to_active_pane(Box::new(sped_page), None, true, window, cx)
    }
}

pub struct SpedPage {
    query_editor: Entity<Editor>,
    version: String,
}

fn helper() -> PyResult<String> {
    Python::with_gil(|py| {
        let module = PyModule::from_code(
            py,
            c_str!(
                r#"
import signal

import pandas as pd

# numpy messes up keyboard interrupts, this restores defaults
signal.signal(signal.SIGINT, signal.SIG_DFL)

def wrapper():
    data = {"col1": ["val1"], "col2": ["val2"]}
    df = pd.DataFrame.from_dict(data)
    return str(df.iloc[0].col1)
"#
            ),
            c_str!("module.py"),
            c_str!("module"),
        )?;

        let version: String = module.getattr("wrapper")?.call0()?.extract()?;
        Ok(version)
    })
}

impl SpedPage {
    pub fn new(window: &mut Window, cx: &mut Context<Workspace>) -> Entity<Self> {
        cx.new(|cx| {
            let query_editor = cx.new(|cx| {
                let mut input = Editor::single_line(window, cx);
                input.set_placeholder_text("Search extensions...", cx);
                input
            });

            let version: String = helper().unwrap();

            let this = Self {
                query_editor,
                version,
            };
            this
        })
    }
}

impl Render for SpedPage {
    fn render(&mut self, _: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        v_flex()
            .size_full()
            .bg(cx.theme().colors().editor_background)
            .child(
                v_flex()
                    .gap_4()
                    .pt_4()
                    .px_4()
                    .bg(cx.theme().colors().editor_background)
                    .child(
                        h_flex()
                            .w_full()
                            .gap_2()
                            .justify_between()
                            .child(Headline::new(&self.version).size(HeadlineSize::XLarge)),
                    ),
            )
    }
}

impl EventEmitter<ItemEvent> for SpedPage {}

impl Focusable for SpedPage {
    fn focus_handle(&self, cx: &App) -> gpui::FocusHandle {
        self.query_editor.read(cx).focus_handle(cx)
    }
}

impl Item for SpedPage {
    type Event = ItemEvent;

    fn tab_content_text(&self, _detail: usize, _cx: &App) -> SharedString {
        "Sped".into()
    }

    fn telemetry_event_text(&self) -> Option<&'static str> {
        Some("Sped Page Opened")
    }

    fn show_toolbar(&self) -> bool {
        false
    }

    fn clone_on_split(
        &self,
        _workspace_id: Option<WorkspaceId>,
        _window: &mut Window,
        _: &mut Context<Self>,
    ) -> Option<Entity<Self>> {
        None
    }

    fn to_item_events(event: &Self::Event, mut f: impl FnMut(workspace::item::ItemEvent)) {
        f(*event)
    }
}
```
