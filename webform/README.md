# ChatLingual Webform Listener

The ChatLingual Webform Listener integrates new and existing website forms with ChatLingual. With some simple configuration, any existing page element can be used to create conversations within ChatLingual.

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
        // API endpoint provided by ChatLingual
        apiEndpoint: '<API_ENDPOINT>',
        // flag that enables verbose console logging
        debug: true | false,
        // webform listener configuration
        webform: {
          // listener config (see below for details)
          listener: { ... },
          // hooks config (see below for details)
          hooks: { ... },
          // form config (see below for details)
          form: { ... },
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

This is the API key provisioned by ChatLingual.

### `chatlingual.apiEndpoint: string`

The API endpoint pointing towards your infrastructure provided by ChatLingual.

### `chatlingual.debug: boolean?`

Instructs the script to output additional debugging logs to the console. This property is useful when embedding this script for the first time, typically in a test page where noisy log output doesn't matter. It's not recommended to set this property to `true` in production environments.

### `chatlingual.webform.listener: Listener?`

```ts
type Listener {
  // https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelector
  querySelector: string;
  // https://developer.mozilla.org/en-US/docs/Web/Events
  event: string;
}
```

When `chatlingual.webform.listener` is provided, the script will automatically attach an event listener to the DOM element found using `querySelector`.

For example, the following block:

```ts
chatlingual.webform.listener = {
  querySelector: "#your-form",
  event: "submit",
};
```

will attach the `submit` event handler to the DOM element with the ID `your-form`. The conversation will be created within ChatLingual when `your-form` is fires the configured event, in this case the `submit` event.

### `chatlingual.webform.hooks: Hooks?`

```ts
type Hooks {
  beforeSubmit: (e?: Event) => void;
  afterSubmit: (e?: Event) => void;
  onError: (err?: any) => void;
}
```

Hooks provide a way to perform logic before or after a form submission or when an error occurs. If the form submission fires from the event listener described by `chatlingual.webform.listener`, the event will be passed into the hook for `beforeSubmit` and `afterSubmit`. This can be used to control the behavior of the submission.

For example, the following block will prevent a form from executing its default behavior when submitted (i.e. refreshing the page):

```ts
chatlingual.webform.listener = {
  querySelector: "#your-form",
  event: "submit",
};

chatlingual.webform.hooks = {
  beforeSubmit: (e) => e.preventDefault(),
};
```

Hooks can be used to accomplish more advanced user experiences, such as toggling loading states in the UI, showing error toasts, etc.

If `chatlingual.webform.listener` is omitted, the `beforeSubmit` and `afterSubmit` do not accept arguments.

### `chatlingual.webform.form: Form`

```ts
type Form {
  // defaults to the browser's language, but can be used to override the browser
  language?: string;
  conversation?: () => Conversation;
  customer?: () => Customer
};

type Conversation {
  // email is the only supported channel at the moment
  channel: 'email';
  // data pertaining to the email conversation that'll be created within ChatLingual
  email: {
    subject: string;
    body: string;
    attachments: File[];
  };
  // optionally,  custom fields that'll populate the Agent Desktop's Conversation Details
  fields?: Record<string, any>;
};

type Customer {
  // che customer's email address
  email: string;
  firstName?: string;
  lastName?: string;
  // optionally, custom fields that'll populate the Agent Desktop's Customer Details
  fields?: Record<string, any>;
}
```

The `chatlingual.webform.form` object has two keys that'll be used to create the data needed to submit the conversation to ChatLingual, `chatlingual.webform.form.conversation()` and `chatlingual.webform.form.conversation()`. Both of these keys are JavaScript functions and are executed when the form is submitted. This provides an opportunity to collect and structure all of the necessary conversation and customer data needed to create a conversation within ChatLingual.

For example, the following block illustrates a configuration that'll collect data from some elements on the page that will be used to create an `email` conversation:

```js
chatlingual.webform.form = {
  conversation: function() {
    return {
      channel: 'email',
      email: {
        body: document.getElementById('body').value,
        subject: document.getElementById('subject').value,
      },
      fields: {
        'custom_field_a': document.getElementById('custom-field-a').value
        'custom_field_b': document.getElementById('custom-field-b').value
      }
    };
  },
  customer: function() {
    return {
      firstName: document.getElementById('first-name').value,
      lastName: document.getElementById('last-name').value,
      email: document.getElementById('email').value,
      fields: {
        'custom_field_c': document.getElementById('custom-field-c').value
      }
    };
  }
};
```

The above example assumes that your page has some elements with the IDs `#body`, `#subject`, `#custom-field-a`, `#custom-field-b`, `#custom-field-c`, `#first-name`, `#last-name`, and `#email`. The Webform Listener isn't opinionated on how your form is structured. It only cares that the `.customer()` and `.conversation()` functions return valid objects set when submitting the form.

You're able to specify custom field sets for both the conversation and the customer by adding the `.fields` key, which needs to return an object whose keys are the field names. ChatLingual provisions the custom fields and can provide the list of custom fields for both the conversation and the customer. In the above example, the conversation has two custom fields: `custom_field_a` and `custom_field_b`. The customer has one custom field: `custom_field_c`.
