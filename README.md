# vue3-file-upload

A Vue 3 component for handling file uploads with features like previews, validations, and progress tracking.

## Features

- Drag and drop file support
- File previews (images, generic icons for others)
- File validation (size, type, MIME type, custom validation)
- Multiple file selection support
- Upload progress tracking
- Error handling
- Customizable appearance via CSS classes and variables
- Slot for custom file previews

## Installation

Install the package using npm:

```bash
npm install vue3-file-upload
```

## Usage

Import and use the component in your Vue files:

```vue
<template>
  <FileUploader :multiple="true" :maxFileSize="5 * 1024 * 1024" // 5MB
  :maxTotalSize="20 * 1024 * 1024" // 20MB :maxFiles="5" :allowedTypes="['.jpg',
  '.png', '.pdf']" :allowedMimeTypes="['image/jpeg', 'image/png',
  'application/pdf']" uploadUrl="/api/upload" @fileSelected="handleFileSelected"
  @fileRemoved="handleFileRemoved" @uploadProgress="handleUploadProgress"
  @uploadSuccess="handleUploadSuccess" @uploadError="handleUploadError"
  @validationError="handleValidationError"
  uploaderClass="my-custom-uploader-class"
  previewClass="my-custom-preview-class" itemClass="my-custom-item-class"
  errorClass="my-custom-error-class" />
</template>

<script>
import FileUploader from "vue3-file-upload";

export default {
  components: {
    FileUploader,
  },
  setup() {
    const handleFileSelected = (files) => {
      console.log("Files selected:", files);
      // files is an array of file objects with added properties like status, progress, previewUrl
    };

    const handleFileRemoved = (file) => {
      console.log("File removed:", file);
      // file is the removed file object
    };

    const handleUploadProgress = ({ file, progress }) => {
      console.log(`Upload progress for ${file.name}: ${progress}%`);
      // file is the file object, progress is a number (0-100)
    };

    const handleUploadSuccess = ({ file, response }) => {
      console.log(`Upload success for ${file.name}:`, response);
      // file is the file object, response is the server response text
    };

    const handleUploadError = ({ file, error, status, response }) => {
      console.error(`Upload error for ${file.name}:`, error, status, response);
      // file is the file object, error is the error message, status is the HTTP status, response is the server response text
    };

    const handleValidationError = (errors) => {
      console.error("Validation errors:", errors);
      // errors is an array of strings describing the validation failures
    };

    return {
      handleFileSelected,
      handleFileRemoved,
      handleUploadProgress,
      handleUploadSuccess,
      handleUploadError,
      handleValidationError,
    };
  },
};
</script>
```

## Props

| Prop               | Type                        | Default | Description                                                                    |
| ------------------ | --------------------------- | ------- | ------------------------------------------------------------------------------ | ------------------ |
| `multiple`         | `Boolean`                   | `false` | Allow multiple file selection.                                                 |
| `maxFileSize`      | `Number`                    | `0`     | Maximum allowed size per file in bytes (0 for no limit).                       |
| `maxTotalSize`     | `Number`                    | `0`     | Maximum allowed total size for all files in bytes (0 for no limit).            |
| `maxFiles`         | `Number`                    | `0`     | Maximum number of files allowed (0 for no limit).                              |
| `allowedTypes`     | `Array<string>`             | `[]`    | Array of allowed file extensions (e.g., `['.jpg', '.png']`). Case-insensitive. |
| `allowedMimeTypes` | `Array<string>`             | `[]`    | Array of allowed MIME types (e.g., `['image/jpeg', 'image/png']`).             |
| `validateFile`     | `Function`                  | `null`  | Custom validation function `(file: File) => boolean                            | Promise<boolean>`. |
| `uploadUrl`        | `String`                    |         | **Required**. The URL endpoint for file uploads.                               |
| `uploaderClass`    | `String`, `Array`, `Object` | `''`    | Custom CSS class(es) for the main uploader container.                          |
| `previewClass`     | `String`, `Array`, `Object` | `''`    | Custom CSS class(es) for the file previews container.                          |
| `itemClass`        | `String`, `Array`, `Object` | `''`    | Custom CSS class(es) for each individual file item.                            |
| `errorClass`       | `String`, `Array`, `Object` | `''`    | Custom CSS class(es) for the error messages container.                         |

## Events

| Event             | Payload                                                             | Description                                                                                |
| ----------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `fileSelected`    | `files: File[]`                                                     | Emitted when files are selected or dropped. Payload is the current list of selected files. |
| `fileRemoved`     | `file: File`                                                        | Emitted when a file is removed. Payload is the removed file object.                        |
| `uploadProgress`  | `{ file: File, progress: number }`                                  | Emitted during file upload to report progress (0-100).                                     |
| `uploadSuccess`   | `{ file: File, response: string }`                                  | Emitted when a file upload is successful.                                                  |
| `uploadError`     | `{ file: File, error: string, status?: number, response?: string }` | Emitted when a file upload fails.                                                          |
| `validationError` | `errors: string[]`                                                  | Emitted when files fail validation. Payload is an array of error messages.                 |

## Slot

The component provides a default slot (`#file-preview`) to allow for custom rendering of file previews. The slot scope provides access to `files` (the array of selected files), `removeFile` (a function to remove a file by index), and `uploadFile` (a function to manually trigger upload for a file).

```vue
<FileUploader ...>
  <template #file-preview="{ files, removeFile, uploadFile }">
    <div v-for="(file, index) in files" :key="file.name + index">
      <!-- Your custom preview logic here -->
      <span>{{ file.name }}</span>
      <button @click="removeFile(index)">Remove</button>
      <button v-if="file.status === 'error'" @click="uploadFile(file, index)">Retry</button>
    </div>
  </template>
</FileUploader>
```

## Styling

The component uses CSS variables for basic theming. You can override these variables or use the provided class props (`uploaderClass`, `previewClass`, `itemClass`, `errorClass`) to apply custom styles.

Example of overriding CSS variables:

```css
.my-custom-uploader-class {
  --file-uploader-border-color: purple;
  --file-uploader-drag-over-border-color: darkorchid;
}
```

Example of adding custom styles via class:

```css
.my-custom-item-class {
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.1);
}
```

Refer to the `FileUploader.vue` source code for the full list of CSS variables.
