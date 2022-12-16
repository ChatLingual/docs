<div><br/></div>
<div><br/></div>

<div align="center">
  <img src="logo.svg" />
  <div>
    <h1>Webform Listener</h1>
  </div>
  The ChatLingual Webform Listener integrates new and existing website forms with ChatLingual. With some simple configuration, any existing page element can be used to create conversations within ChatLingual.
</div>

<div><br/></div>
<div><br/></div>

## Getting Started

First, the ChatLingual configuration object needs to be declared as a global variable (i.e. a variable attached to the global `window` object that can be accessed from the main JavaScript thread). Second, the Webform Listener script must be embedded within the page that includes the form. Scripts are typically embedded within the `<head>` tag, but this may vary depending on your page's implementation.

For example, the following HTML shows how one might structure this at a high-level:

```html
<html>
  <head>
    <!-- global ChatLingual configuration -->
    <script>
      chatlingual = {
        // API key provided by ChatLingual
        apiKey: '<API_KEY>',
        // flag that enables verbose console logging
        debug: true | false,
        // webform listener configuration
        webform: {
          // API endpoint provided by ChatLingual
          endpoint: '<ENDPOINT>',
          // optionally, the customer's language override
          language: '<LANGUAGE_CODE>',
          // configurations (see details below)
          configs: {
            ['submitEmail' | 'submitEmailRepy']: {
              // optionally, an object that lets ChatLingual respond to configured
              // events on an element on the page
              listener: { ... },
              // optionally, a set of functions that fire at different points during
              // the lifecycle of a form submission
              hooks: { ... },
              // function that returns customer information when called
              customer: () => { ... },
              // function that returns conversation information when called
              conversation: () => { ... },
              // function that returns email information when called
              email: () => { ... },
            }
          }
        },
      };
    </script>
    <!-- this must come after the global configuration above
    please contact ChatLingual for script URL -->
    <script src="<WEBFORM_SCRIPT_URL>"></script>
  </head>
  <body>
    ...
    <form id="your-form">...</form>
    ...
  </body>
</html>
```

## Configuration

This section breaks down the ChatLingual configuration object (`window.chatlingual`). The console will report invalid configurations most of the time. If you have issues implementing the configuration, or have specific questions, please [email support](mailto:help@chatlingual.com).

### `chatlingual.apiKey: string`

The API key provisioned by ChatLingual.

### `chatlingual.debug: boolean?`

Instructs the script to output additional debugging logs to the console. This property is useful when embedding this script for the first time, typically in a test page where noisy log output doesn't matter. It's not recommended to set this property to `true` in production environments.

### `chatlingual.webform.endpoint: string`

The API endpoint provided by ChatLingual.

### `chatlingual.webform.configs: Record<WebformSubmitType, WebformConfig>`

An object, whose keys are the `WebformSubmitType` (either `'submitEmail'`, or `'submitEmailReply'`) and values are configuration settings.

Use `'submitEmailReply'` for following up with existing conversations, typically via submitting a response to an existing ticket through a form.

The data requirements (i.e. the objects returned by `.email()` and `.conversation()`, and `.customer()`) differ based on this type. See below for information on required fields:

### `chatlingual.webform.configs[type].listener: Listener?`

```ts
type Listener {
  // https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelector
  querySelector: string;
  // https://developer.mozilla.org/en-US/docs/Web/Events
  event: string;
}
```

When a listener is provided, the script will automatically attach an event listener to the DOM element found using `querySelector`.

For example, the following block:

```ts
const listener = {
  querySelector: "#your-form",
  event: "submit",
};
```

will attach the `submit` event handler to the DOM element with the ID `your-form`. The conversation will be created within ChatLingual when `your-form` fires the configured event (in this case the `submit` event).

The listener provides a convenient way to declaratively describe form behavior but is not required. For advanced use cases, where you might need granular control over _when_ or _how_ the form is submitted, you can call the submit functions directly. The script attaches both `submitEmail` and `submitEmailReply` directly to the global ChatLingual configuration:

```ts
window.chatlingual.webform.submitEmail();
window.chatlingual.webform.submitEmailReply();
```

For example, this code block will submit an email when `customButton` is clicked.

```ts
customButton.addEventListener("click", window.chatlingual.webform.submitEmail);
```

### `chatlingual.webform.configs[type].hooks: Hooks?`

```ts
type Hooks {
  beforeSubmit: (e?: Event) => void;
  afterSubmit: (response: any, e?: Event) => void;
  onError: (err?: any) => void;
}
```

Hooks provide a way to perform logic before or after a form submission or when an error occurs. When a listener is provided, the event will be provided to `beforeSubmit()`'s first argument and `afterSubmit()`'s second argument. When a listener is omitted, the `beforeSubmit()` and `afterSubmit()` hooks don't accept any arguments.

As an example, the following block will prevent a form from executing its default behavior when submitted (i.e. refreshing the page):

```ts
const listener = {
  querySelector: "#your-form",
  event: "submit",
};

const hooks = {
  // prevent the default form submit action
  beforeSubmit: (e) => {
    // `e` here is only provided if the submission was called using an event listener
    if (e) {
      e.preventDefault();
    }
  },
};
```

Hooks can be used to accomplish more advanced user experiences, such as toggling loading states in the UI, showing error toasts, etc. Refer to this [example](example.html) to see how hooks can be used.

### `chatlingual.webform.configs[type].conversation: () => Conversation`

The `conversation()` function returns an object that's sent along to ChatLingual. The object returned by this is specific to the conversation, and the fields depend on the type of form submission. There aren't any required fields when the submission type is `submitEmail`, but either the `id` or `externalId` properties are required when the submission type is `submitEmailReply`.

```ts
type Conversation {
  // the ChatLingual conversation ID (either this or `externalId`) is required
  // when the submission type is 'submitEmailReply'
  id: string;
  // the ID of an external system that's associated with the ChatLingual
  // conversation, i.e. a ticket number from a CRM
  externalId: string;
  // optionally,  custom fields that'll populate ChatLingual's conversation details
  fields?: Record<string, any>;
};
```

For `submitEmailReply` ChatLingual will either use `id` or `externalId` to look up the existing conversation. If found, a response will be submitted to that existing conversation.

### `chatlingual.webform.configs[type].customer: () => Customer`

```ts
type Customer {
  // the customer's email address
  email: string;
  firstName?: string;
  lastName?: string;
  // optionally, custom fields that'll populate ChatLingual's customer details
  fields?: Record<string, any>;
}
```

### `chatlingual.webform.configs[type].email: () => Email`

```ts
type Email {
  // the subject is only required when the submission type is `submitEmail`,
  // for `submitEmailReply` this can be omitted
  subject: string;
  body: string;
  attachments: File[];
}
```

This object structures the actual email that'll be sent.

## Debugging

It's recommended to set `window.chatlingual.debug` to `true` when configuring this for the first time.

It's also possible to inspect the data before form submissions are called using the [Dev Tools console](https://developer.chrome.com/docs/devtools/console/).

The following functions can be invoked before submission to check whether they're returning the correct data.

```ts
window.chatlingual.webform.configs.submitEmail.conversation();
window.chatlingual.webform.configs.submitEmail.customer();
window.chatlingual.webform.configs.submitEmail.email();
window.chatlingual.webform.configs.submitEmailReply.conversation();
window.chatlingual.webform.configs.submitEmailReply.customer();
window.chatlingual.webform.configs.submitEmailReply.email();
```

The script will also report errors to the console if the data isn't structured correctly. In addition, the `onError` hook will be invoked with invalid data, which can be used for displaying errors to the end user.

## Example

Refer to this [example](example.html) for a full implementation.

## Questions

Please [reach out to us](help@chatlingual.com) with any questions on implementation or best practices.
