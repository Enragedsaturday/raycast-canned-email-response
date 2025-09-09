# Canned Replies Raycast Extension

Phase 1: Project Setup

Initialize the Extension: Create a package.json manifest for the Raycast
extension with basic metadata, dependencies, and scripts. Include the extension
title, description, author, license, categories, and a command entry for the main view. Set the main command’s name to "index" (mapping to src/index.tsx) and give it a default keyboard shortcut of Option+R (⌥R). Use the Raycast API and utilities packages and set up scripts for development, linting, and type-checking.

{
  "name": "canned-replies",
  "title": "Canned Replies",
  "description": "Manage a list of canned email replies and quickly insert them in Apple Mail.",
  "version": "1.0.0",
  "author": "your-username",
  "license": "MIT",
  "categories": [
    "Productivity",
    "Applications"
  ],
  "commands": [
    {
      "name": "index",
      "title": "Canned Replies",
      "description": "Search and insert saved email replies",
      "mode": "view",
      "shortcut": {
        "key": "r"
        "modifiers": [
          "opt"
        ]
      }
    }
  ],
  "dependencies": {
    "@raycast/api": "^1.102.0",
    "@raycast/utils": "^2.2.0"
  },
  "devDependencies": {
    "@raycast/eslint-config": "^2.0.4",
    "@types/node": "^18.0.0",
    "@types/react": "^17.0.0",
    "eslint": "^8.42.0",
    "typescript": "^4.5.4"
  },
  "scripts": {
    "dev": "raycast develop",
    "build": "raycast build",
    "lint": "raycast lint",
    "type-check": "tsc --noEmit"
  }
}


Scaffold the Main Command: In the src directory, create a file index.tsx. Export a React component that renders an empty Raycast List. This will be the main view of the extension (with a search bar placeholder indicating to search replies). For now, no list items are shown (we will add functionality in later phases).

// src/index.tsx
import { List } from "@raycast/api";

export default function Command() {
  return <List searchBarPlaceholder="Search canned replies..." />;
}


Install and Verify: Run npm install to install dependencies. Then run npm run dev to start the extension in development mode. Open Raycast and trigger the "Canned Replies" command (you can press ⌥R as configured). You should see an empty list view with no errors. Also, run npm run lint and npm run type-check to ensure the project has no linting or type errors at this stage.

Phase 2: Local Storage and List UI

Define Data Model: At the top of src/index.tsx, define a TypeScript interface for a canned reply template. Each template will have an id, title, body, and timestamp fields createdAt/updatedAt (ISO date strings). Use the Raycast LocalStorage (via the useLocalStorage hook from @raycast/utils) to store an array of these templates. Initialize the storage with an empty array. The hook provides value (current list of templates) and a setValue function to update it.

Render List Items: Update the Command component to display saved replies. If the list is loading or empty, show a helpful EmptyView. Otherwise, map each saved template to a List.Item showing its title (and a snippet of the body as a subtitle for context). Include a generic document icon for each item. Prepare an ActionPanel for list items (to be filled in upcoming phases). Also, add an EmptyView action panel so users can create a new template when none exist (we’ll implement creation next).

// src/index.tsx (updated)
import { List, ActionPanel, Action, Icon } from "@raycast/api";
import { useLocalStorage } from "@raycast/utils";

interface CannedReply {
  id: string;
  title: string;
  body: string;
  createdAt: string;
  updatedAt: string;
}

export default function Command() {
  const { value: replies, setValue: setReplies, isLoading } = useLocalStorage<CannedReply[]>("canned-replies", []);

  return (
    <List isLoading={isLoading} searchBarPlaceholder="Search canned replies...">
      {replies && replies.length > 0 ? (
        replies.map((reply) => (
          <List.Item
            key={reply.id}
            icon={Icon.TextDocument}
            title={reply.title}
            subtitle={reply.body.slice(0, 50)}
            actions={
              <ActionPanel>
                {/* Template-specific actions will be added in later phases */}
                <ActionPanel.Section title="General">
                  <Action.Push title="Create New Template" icon={Icon.Plus} target={<></>} /> 
                </ActionPanel.Section>
              </ActionPanel>
            }
          />
        ))
      ) : (
        <List.EmptyView
          icon={Icon.TextDocument}
          title="No Canned Replies"
          description="No templates found. Create a new reply template to get started."
          actions={
            <ActionPanel>
              <Action.Push title="Create New Template" icon={Icon.Plus} target={<></>} />
            </ActionPanel>
          }
        />
      )}
    </List>
  );
}


(In the code above, the Action.Push targets are left empty (<></>) as placeholders – we will replace these with actual components in the next phase.)

Test the List UI: Run npm run dev again. The "Canned Replies" view should now show a "No Canned Replies" placeholder message. The list is empty, and a "Create New Template" action is visible. Verify there are no runtime errors. Also run npm run lint and npm run type-check to confirm the code is clean.

Phase 3: Creating New Templates

Create the Template Form Component: Add a new file src/TemplateForm.tsx. This component will present a form for creating a new canned reply. Use Raycast Form fields for the title and body. On form submission, generate a unique id (using crypto.randomUUID() for strong uniqueness) and timestamp fields. Update the local storage by appending the new template to the existing list (using the setReplies function). After saving, show a success Toast and return to the main list view by popping the navigation. Include basic validation: if the title is empty, show an error Toast and prevent submission.

// src/TemplateForm.tsx
import { Form, ActionPanel, Action, popToRoot, showToast, Toast } from "@raycast/api";
import { useLocalStorage } from "@raycast/utils";
import { useState } from "react";
import { randomUUID } from "crypto";
import { CannedReply } from "./index";

export default function TemplateForm() {
  const { value: replies, setValue: setReplies } = useLocalStorage<CannedReply[]>("canned-replies", []);
  const [title, setTitle] = useState("");
  const [body, setBody] = useState("");

  async function handleSubmit(values: { title: string; body: string }) {
    const titleText = values.title.trim();
    if (titleText.length === 0) {
      await showToast({ style: Toast.Style.Failure, title: "Title cannot be empty" });
      return false; // Keep the form open for correction
    }
    const newTemplate: CannedReply = {
      id: randomUUID(),
      title: titleText,
      body: values.body,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString()
    };
    await setReplies([...(replies || []), newTemplate]);
    await showToast({ style: Toast.Style.Success, title: "Template Saved" });
    popToRoot(); // Close form and return to list
  }

  return (
    <Form 
      navigationTitle="New Canned Reply"
      actions={
        <ActionPanel>
          <Action.SubmitForm title="Create Template" onSubmit={handleSubmit} />
        </ActionPanel>
      }
    >
      <Form.TextField id="title" title="Title" placeholder="Template title" value={title} onChange={setTitle} autoFocus />
      <Form.TextArea id="body" title="Body" placeholder="Email reply text..." value={body} onChange={setBody} />
    </Form>
  );
}


Integrate the Create Action: In src/index.tsx, import the new TemplateForm component. Update the previously empty <Action.Push> targets to push this form. Add a keyboard shortcut (for example, ⌘+N) so users can quickly open the new template form. The "Create New Template" action should be available both when the list is empty and in the list’s action panel for existing items.

// src/index.tsx (excerpt – integrate TemplateForm for creation)
import TemplateForm from "./TemplateForm";
// ... inside List.Item actions and EmptyView actions:
<Action.Push 
  title="Create New Template" 
  icon={Icon.Plus} 
  target={<TemplateForm />} 
  shortcut={{ modifiers: ["cmd"], key: "n" }} 
/>


Try Creating a Template: Reload the extension with npm run dev. In Raycast, trigger "Canned Replies" and use the "Create New Template" action (or press ⌘+N). In the form, enter a sample title and body, then submit. The form should close, and the main list will now display the new template in the list. Verify that the template appears with the correct title and a subtitle snippet of the body. The local storage has been updated. Run npm run lint and npm run type-check to ensure everything remains clean.

Phase 4: Editing and Duplicating Templates

Extend TemplateForm for Reuse: Modify src/TemplateForm.tsx to support both editing and duplicating existing templates in addition to creating new ones. Add optional props to TemplateForm (e.g. an existing template and a boolean duplicate flag). If an existing template is provided, initialize the form fields with its title and body. The form’s navigation title and submit button text should reflect the context (e.g. "Edit Template" vs "Create Template"). In the submit handler:

If editing (duplicate flag is false), find the item in storage by ID and update its title/body and updatedAt timestamp.

If duplicating (duplicate flag is true) or no existing template is provided, create a new entry (with a new ID and timestamps) as in Phase 3.

// src/TemplateForm.tsx (updated for edit/duplicate)
type TemplateFormProps = {
  existing?: CannedReply;
  duplicate?: boolean;
};

export default function TemplateForm(props: TemplateFormProps) {
  const { value: replies, setValue: setReplies } = useLocalStorage<CannedReply[]>("canned-replies", []);
  const isEdit = props.existing && !props.duplicate;
  const [title, setTitle] = useState<string>(props.existing ? props.existing.title + (props.duplicate ? " (Copy)" : "") : "");
  const [body, setBody] = useState<string>(props.existing ? props.existing.body : "");

  async function handleSubmit(values: { title: string; body: string }) {
    const titleText = values.title.trim();
    if (titleText.length === 0) {
      await showToast({ style: Toast.Style.Failure, title: "Title cannot be empty" });
      return false;
    }
    if (isEdit && props.existing) {
      // Update existing template
      const updatedList = (replies || []).map((item) =>
        item.id === props.existing!.id
          ? { ...item, title: titleText, body: values.body, updatedAt: new Date().toISOString() }
          : item
      );
      await setReplies(updatedList);
      await showToast({ style: Toast.Style.Success, title: "Template Updated" });
    } else {
      // Create a new template (for new or duplicated)
      const newTemplate: CannedReply = {
        id: randomUUID(),
        title: titleText,
        body: values.body,
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString()
      };
      await setReplies([...(replies || []), newTemplate]);
      await showToast({ style: Toast.Style.Success, title: props.duplicate ? "Template Duplicated" : "Template Created" });
    }
    popToRoot();
  }

  return (
    <Form 
      navigationTitle={isEdit ? "Edit Canned Reply" : "Canned Reply"}
      actions={
        <ActionPanel>
          <Action.SubmitForm title={isEdit ? "Save Changes" : "Save Template"} onSubmit={handleSubmit} />
        </ActionPanel>
      }
    >
      <Form.TextField id="title" title="Title" value={title} onChange={setTitle} />
      <Form.TextArea id="body" title="Body" value={body} onChange={setBody} />
    </Form>
  );
}


Add Edit and Duplicate Actions: In src/index.tsx, update the ActionPanel for each List.Item to include Edit and Duplicate actions. Use Action.Push to navigate to TemplateForm with appropriate props:

Edit Template: Push TemplateForm with existing={reply} (the selected template) to allow in-place editing.

Duplicate Template: Push TemplateForm with existing={reply} and duplicate=true. This will pre-fill the form with the template’s content (with “ (Copy)” appended to the title) and on submit will save a new template.
Assign distinct icons and keyboard shortcuts if desired (e.g. ⌘+E for edit, ⌘+D for duplicate).

// src/index.tsx (excerpt – add Edit & Duplicate actions)
<List.Item
  // ...other props
  actions={
    <ActionPanel>
      <ActionPanel.Section title="Manage Template">
        <Action.Push title="Edit Template" icon={Icon.Pencil} target={<TemplateForm existing={reply} />} shortcut={{ modifiers: ["cmd"], key: "e" }} />
        <Action.Push title="Duplicate Template" icon={Icon.TextDocument} target={<TemplateForm existing={reply} duplicate={true} />} shortcut={{ modifiers: ["cmd", "shift"], key: "d" }} />
        {/* Delete action will be added in the next phase */}
      </ActionPanel.Section>
      <ActionPanel.Section title="General">
        <Action.Push title="Create New Template" icon={Icon.Plus} target={<TemplateForm />} shortcut={{ modifiers: ["cmd"], key: "n" }} />
        {/* Import/Export actions to be added later */}
      </ActionPanel.Section>
    </ActionPanel>
  }
/>


Test Edit and Duplicate: Reload with npm run dev. Add a couple of test templates if not already present. Now select an existing item and choose "Edit Template" (or press the shortcut). Change some text and submit – the list should update that item’s title/body. Next, try "Duplicate Template" on an item – a new form opens with the same content (and “(Copy)” in the title). Submitting it should create a new template entry (with a new ID) in the list. Ensure the list now contains the duplicated entry alongside the original. Run npm run lint and npm run type-check to catch any issues.

Phase 5: Deleting Templates

Add Delete Action with Confirmation: For each template in the list, include a Delete action. This action should prompt the user for confirmation (because deletion is irreversible). Use Raycast’s confirmAlert API to show an alert. If the user confirms, remove the template from the local storage list and show a success Toast. If they cancel, do nothing. Mark the delete action as destructive in the UI.

Implement the delete handler: In src/index.tsx, import confirmAlert and define an async function to delete a template by id. Use setReplies to filter out the deleted item.

Add the action to the UI: In the ActionPanel’s "Manage Template" section, add an Action calling this handler. Give it a red trash icon and a descriptive title.

// src/index.tsx (excerpt – add delete functionality)
import { confirmAlert, Alert, showToast, Toast } from "@raycast/api";
// ...

export default function Command() {
  const { value: replies, setValue: setReplies } = useLocalStorage<CannedReply[]>("canned-replies", []);

  async function deleteTemplate(id: string) {
    const ok = await confirmAlert({
      title: "Delete Template",
      message: "Are you sure you want to delete this canned reply?",
      primaryAction: { title: "Delete", style: Alert.ActionStyle.Destructive },
      dismissAction: { title: "Cancel" }
    });
    if (!ok) return;
    const newList = (replies || []).filter((item) => item.id !== id);
    await setReplies(newList);
    await showToast({ style: Toast.Style.Success, title: "Template deleted" });
  }

  // ... inside <ActionPanel.Section title="Manage Template"> for each List.Item:
  <Action 
    title="Delete Template" 
    icon={Icon.Trash} 
    style={Action.Style.Destructive} 
    onAction={() => deleteTemplate(reply.id)} 
  />
}


Verify Deletion: Restart the dev session (npm run dev). Create a couple of sample templates if needed. Select a template and use the "Delete Template" action. A confirmation alert should appear. If you choose "Delete", the item is removed from the list. If you choose "Cancel", the list remains unchanged. Ensure the Toast messages appear appropriately. Run npm run lint and npm run type-check after this update to confirm everything passes.

Phase 6: Importing Templates from JSON

Build the Import Form: To allow importing templates from a JSON file, create a new component src/ImportForm.tsx. This form will have a File Picker field to choose a JSON file. On submit, it should read and parse the file, then replace the current templates in local storage with the imported ones. Include robust error handling:

If the file content is not valid JSON or doesn’t match the expected format, show an error Toast.

If the extension already has templates stored, confirm with the user before overwriting them (e.g. “This will replace all existing templates. Continue?”).

Auto-fill missing fields if possible (generate new IDs or timestamps for imported items missing them). Skip or default any malformed entries.

After successful import, show a success Toast and pop back to the list.

// src/ImportForm.tsx
import { Form, ActionPanel, Action, showToast, Toast, confirmAlert, Alert, popToRoot } from "@raycast/api";
import { useLocalStorage } from "@raycast/utils";
import { promises as fs } from "fs";
import { CannedReply } from "./index";

export default function ImportForm() {
  const { value: replies, setValue: setReplies } = useLocalStorage<CannedReply[]>("canned-replies", []);

  async function handleImport(values: { file: string[] }) {
    const filePath = values.file[0];
    try {
      // Read and parse the file content
      const content = await fs.readFile(filePath, "utf-8");
      const data = JSON.parse(content);
      if (!Array.isArray(data)) {
        throw new Error("JSON file format is invalid (expected an array of templates).");
      }
      // Confirm override if templates already exist
      if (replies && replies.length > 0) {
        const ok = await confirmAlert({
          title: "Import Templates",
          message: "Importing will replace all existing templates. Continue?",
          primaryAction: { title: "Import", style: Alert.ActionStyle.Destructive },
          dismissAction: { title: "Cancel" }
        });
        if (!ok) {
          popToRoot();
          return;
        }
      }
      // Transform and validate imported data
      const importedList: CannedReply[] = data.map((item) => {
        const title = typeof item.title === "string" && item.title.trim().length > 0 ? item.title : "Untitled";
        const body = typeof item.body === "string" ? item.body : "";
        return {
          id: typeof item.id === "string" && item.id ? item.id : Math.random().toString(36).slice(2),
          title,
          body,
          createdAt: item.createdAt ? String(item.createdAt) : new Date().toISOString(),
          updatedAt: item.updatedAt ? String(item.updatedAt) : new Date().toISOString()
        };
      });
      await setReplies(importedList);
      await showToast({ style: Toast.Style.Success, title: "Import successful", message: `${importedList.length} templates imported` });
      popToRoot();
    } catch (error) {
      console.error("Import failed:", error);
      await showToast({ style: Toast.Style.Failure, title: "Import Failed", message: error instanceof Error ? error.message : String(error) });
    }
  }

  return (
    <Form 
      navigationTitle="Import Templates"
      actions={
        <ActionPanel>
          <Action.SubmitForm title="Import from JSON" onSubmit={handleImport} />
        </ActionPanel>
      }
    >
      <Form.FilePicker id="file" title="JSON File" allowMultipleSelection={false} canChooseFiles canChooseDirectories={false} />
    </Form>
  );
}


Integrate Import Action: In src/index.tsx, import ImportForm and add an "Import from JSON" action. This action should be accessible from the main list view (e.g. in the "General" section of the ActionPanel, and also in the empty state). Use an appropriate icon (e.g. a download arrow) to represent importing.

// src/index.tsx (excerpt – add Import action)
import ImportForm from "./ImportForm";
// ...
<Action.Push title="Import from JSON" icon={Icon.Download} target={<ImportForm />} />


Include the Import action in the ActionPanel:

In the EmptyView actions (so if the list is empty, users can import a backup JSON).

In the main "General" section for when items exist.

For example, your EmptyView actions might have Create and Import, and your General section might include Import (and we will add Export in the next phase).

Test Importing: Prepare a sample JSON file on your disk (for example, export the current templates in the next phase, or create a JSON manually). It should be an array of objects with title and body (and optionally id, createdAt, updatedAt). In Raycast, choose "Import from JSON" and select the file. Confirm the overwrite if prompted. After importing, the list should update to show the templates from the file. Try importing invalid JSON or a wrong format to ensure the error Toast appears. Run npm run lint and npm run type-check to catch any mistakes.

Phase 7: Exporting Templates to JSON

Build the Export Form: Create src/ExportForm.tsx for exporting current templates to a JSON file. The form will have a Directory Picker (to choose a folder for saving the JSON). On submit, gather the current templates from local storage and write them to a file in the chosen directory. Use Node’s fs module to create the file (name it something like "canned-replies.json"). Handle potential errors (e.g. no write permission) with a failure Toast. If there are no templates to export, warn the user and abort.

// src/ExportForm.tsx
import { Form, ActionPanel, Action, showToast, Toast, popToRoot } from "@raycast/api";
import { useLocalStorage } from "@raycast/utils";
import { promises as fs } from "fs";
import path from "path";
import { CannedReply } from "./index";

export default function ExportForm() {
  const { value: replies } = useLocalStorage<CannedReply[]>("canned-replies", []);

  async function handleExport(values: { directory: string[] }) {
    if (!replies || replies.length === 0) {
      await showToast({ style: Toast.Style.Failure, title: "No templates to export" });
      return false;
    }
    const dirPath = values.directory[0];
    const filePath = path.join(dirPath, "canned-replies.json");
    try {
      await fs.writeFile(filePath, JSON.stringify(replies, null, 2), "utf-8");
      await showToast({ style: Toast.Style.Success, title: "Export successful", message: `Saved to ${filePath}` });
      popToRoot();
    } catch (error) {
      console.error("Export failed:", error);
      await showToast({ style: Toast.Style.Failure, title: "Export Failed", message: error instanceof Error ? error.message : String(error) });
    }
  }

  return (
    <Form 
      navigationTitle="Export Templates"
      actions={
        <ActionPanel>
          <Action.SubmitForm title="Export to JSON" onSubmit={handleExport} />
        </ActionPanel>
      }
    >
      <Form.FilePicker id="directory" title="Save Location" allowMultipleSelection={false} canChooseDirectories canChooseFiles={false} />
    </Form>
  );
}


Integrate Export Action: Import ExportForm in src/index.tsx and add an "Export to JSON" action to the ActionPanel (in the general section). Use a corresponding icon (e.g. an upload arrow). We will not show this action in the empty state (since there’s nothing to export if no templates exist).

// src/index.tsx (excerpt – add Export action)
import ExportForm from "./ExportForm";
// ...
<Action.Push title="Export to JSON" icon={Icon.Upload} target={<ExportForm />} />


Add this to the ActionPanel’s "General" section alongside the Import action. For example:

<ActionPanel.Section title="General">
  <Action.Push title="Create New Template" icon={Icon.Plus} target={<TemplateForm />} shortcut={{ modifiers: ["cmd"], key: "n" }} />
  <Action.Push title="Import from JSON" icon={Icon.Download} target={<ImportForm />} />
  <Action.Push title="Export to JSON" icon={Icon.Upload} target={<ExportForm />} />
</ActionPanel.Section>


Test Exporting: Run npm run dev and open the "Canned Replies" extension. Ensure you have a few templates in the list. Trigger "Export to JSON" and pick a directory (e.g. Desktop or a test folder). After submission, check that a file canned-replies.json appears in the chosen directory containing the templates array. If you try to export with no templates, it should warn "No templates to export". Verify that no errors occur. Run npm run lint and npm run type-check one more time.

Phase 8: Inserting Replies into Apple Mail

Implement Insert & Send: The core feature is to prepend a canned reply’s text into the currently open Apple Mail reply draft. We will add two actions for each list item:

Insert and Send – paste the template text at the top of the draft and immediately send the email.

Insert (No Send) – paste the text at the top of the draft without sending, in case the user wants to review or add more content.

To achieve this, we’ll use AppleScript via Raycast’s runAppleScript utility:

First, call closeMainWindow() to dismiss the Raycast window (so Mail can be frontmost).

Use an AppleScript that activates Mail, sends a ⌘+↑ (Command + Up Arrow) keystroke to move the cursor to the top of the message, sets the clipboard to the template text, pastes it (⌘+V), and if sending, triggers the send shortcut (⌘+⇧+D).

Wrap this in a try/catch and use Toasts to notify success or failure.

Add the insert handler: In src/index.tsx, import runAppleScript and closeMainWindow. Define an async function insertReply(text, send) that performs the above steps. Use the AppleScript “on run” handler to accept the text and a flag, and execute the keystrokes. Ensure any AppleScript errors are caught and reported.

Add actions to UI: In the ActionPanel for each List.Item, create a new section (e.g. "Insert Reply"). Add an Action for "Insert and Send" (default action, triggered by pressing Enter) and another for "Insert without Sending" (triggered by Shift+Enter as an alternate). These actions call insertReply(reply.body, true) or insertReply(reply.body, false) respectively.

// src/index.tsx (excerpt – insert into Mail functionality)
import { runAppleScript, closeMainWindow } from "@raycast/api";
// ...
async function insertReply(bodyText: string, sendNow: boolean) {
  try {
    await closeMainWindow();  // hide Raycast
    const script = `
      on run {textToInsert, sendFlag}
        tell application "Mail" to activate
        tell application "System Events"
          key code 126 using {command down} -- Cmd + Up, go to top of draft
          set the clipboard to textToInsert
          keystroke "v" using {command down} -- paste
          if sendFlag is "1" then
            keystroke "d" using {command down, shift down} -- Cmd + Shift + D to send
          end if
        end tell
      end run
    `;
    await runAppleScript(script, [bodyText, sendNow ? "1" : "0"]);
    const successMessage = sendNow ? "Inserted and sent!" : "Inserted into draft";
    await showToast({ style: Toast.Style.Success, title: successMessage });
  } catch (error) {
    console.error("Insert failed:", error);
    await showToast({ style: Toast.Style.Failure, title: "Failed to insert reply", message: error instanceof Error ? error.message : String(error) });
  }
}
// ... inside <ActionPanel> for each List.Item, add a new section:
<ActionPanel.Section title="Insert Reply">
  <Action title="Insert and Send" onAction={() => insertReply(reply.body, true)} />
  <Action title="Insert without Sending" shortcut={{ modifiers: ["shift"], key: "enter" }} onAction={() => insertReply(reply.body, false)} />
</ActionPanel.Section>


Usage Notes: To use these actions, the user must have an Apple Mail reply or compose window open and focused. The extension will insert the text at the very top of the email body (above any quoted text or signature, since we simulate pressing Command+Up first). The "Insert and Send" action will immediately send the email after inserting the text – on using this, the draft window will close (Mail sends the message). The "Insert without Sending" allows the user to review or edit before sending manually.

Final Testing: Run the extension in dev mode one last time with npm run dev. Open Apple Mail, compose a reply (so that the reply draft window is open and active). In Raycast, trigger "Canned Replies" and select a template. Press Enter for "Insert and Send" – the canned text should appear at the top of the email, and then the email should send (the Mail window will close). Try again with another template but press Shift+Enter for "Insert without Sending" – the text should insert at the top, but the email remains open (not sent). Verify that multiple inserts stack at the top if you trigger it repeatedly (the requirement is always to prepend at top). Ensure all Toast messages appear appropriately for success or any errors (for example, if Mail isn’t open or no draft is focused, the AppleScript may fail – our error Toast will notify the user).

Development Complete: At this point, the extension should compile with no errors and pass linting. Run npm run lint and npm run type-check one final time – the code should be clean. You can now build the extension with npm run build and use it to manage canned replies: creating, editing, organizing (via duplicate/delete), importing/exporting backups, and quickly inserting replies into Apple Mail. The project is ready for use and further refinement as needed.
