# Remodl RTE Features & Implementation Ideas

## Current Features
- Rich text editing with customizable toolbar
- HTML/JSON structure conversion
- Document state management
- Enable/disable functionality with form interaction
- Real-time Firestore sync

## Planned Features

### Document Templates & Firestore Integration
Templates can be mapped to Firestore documents, allowing for:
- Two-way sync between document content and Firestore
- Automatic document generation from templates
- Field mapping and validation

Example implementation:
<div class="work-order" data-remodl="workOrder" data-remodl-ref="workorders/{{id}}">
  <h1 data-remodl-field="title">{{title}}</h1>
  <div class="client-info" data-remodl-field="client">
    <p>Client: {{client.name}}</p>
    <p>Address: {{client.address}}</p>
  </div>
</div>

### Task Integration
Checkboxes and form elements can be linked to Firestore documents:
<input type="checkbox" 
       data-remodl-task-ref="tasks/taskId123" 
       data-remodl-field="status" 
       data-remodl-value-checked="completed"
       data-remodl-value-unchecked="pending">

### Loading States
- Lottie animations for loading/saving states
- Visual feedback for sync status
- Progress indicators for document operations

### Sam Integration
- Document generation from templates
- Field-specific AI assistance
- Content validation and suggestions

## Implementation Details

### Template Processing
Future<String> applyTemplateToDocument({
  required String templateHtml,
  required DocumentReference documentRef,
}) async {
  try {
    final doc = await documentRef.get();
    final data = doc.data() as Map<String, dynamic>;
    
    // Replace placeholders with actual data
    String processedHtml = templateHtml;
    
    // Replace simple fields
    processedHtml = processedHtml.replaceAll('{{id}}', documentRef.id);
    processedHtml = processedHtml.replaceAll('{{title}}', data['title']);
    
    // Handle nested objects
    if (data['client'] != null) {
      processedHtml = processedHtml.replaceAll(
        '{{client.name}}', 
        data['client']['name']
      );
      processedHtml = processedHtml.replaceAll(
        '{{client.address}}', 
        data['client']['address']
      );
    }

    // Add data attributes for two-way sync
    processedHtml = processedHtml.replaceAll(
      'data-remodl-ref="workorders/{{id}}"',
      'data-remodl-ref="workorders/${documentRef.id}"'
    );

    return processedHtml;
  } catch (e) {
    print('Error applying template: $e');
    return templateHtml;
  }
}

### Two-way Sync
void setupDocumentSync() {
  controller.callbacks.onChangeContent = (content) async {
    if (content == null) return;
    
    // Parse HTML to find document reference
    final document = parse(content);
    final docDiv = document.querySelector('[data-remodl]');
    if (docDiv == null) return;
    
    final docRef = docDiv.getAttribute('data-remodl-ref');
    if (docRef == null) return;

    // Find all fields that need syncing
    final syncFields = document.querySelectorAll('[data-remodl-field]');
    
    // Create update map
    Map<String, dynamic> updates = {};
    for (final field in syncFields) {
      final fieldName = field.getAttribute('data-remodl-field');
      if (fieldName != null) {
        updates[fieldName] = field.text;
      }
    }

    // Update Firestore
    try {
      await FirebaseFirestore.instance.doc(docRef).update({
        ...updates,
        'last_modified': FieldValue.serverTimestamp(),
        'last_modified_by': FFAppState().currentUser?.id,
      });
    } catch (e) {
      print('Error syncing document: $e');
    }
  };
}

### Task Updates
void handleTaskUpdate(String taskRef, String field, dynamic value) async {
  try {
    final taskDoc = FirebaseFirestore.instance.doc(taskRef);
    await taskDoc.update({
      field: value,
      'last_modified': FieldValue.serverTimestamp(),
      'last_modified_by': FFAppState().currentUser?.id,
    });
  } catch (e) {
    print('Error updating task: $e');
  }
}

### Document Event Handling in JavaScript
// Add to document.html
document.addEventListener('change', function(e) {
  if (e.target.matches('input[type="checkbox"][data-remodl-task-ref]')) {
    const taskRef = e.target.getAttribute('data-remodl-task-ref');
    const field = e.target.getAttribute('data-remodl-field');
    const value = e.target.checked ? 
      e.target.getAttribute('data-remodl-value-checked') : 
      e.target.getAttribute('data-remodl-value-unchecked');
    
    postChannelMessage({
      'type': 'toDart: taskUpdate',
      'taskRef': taskRef,
      'field': field,
      'value': value,
      'checked': e.target.checked
    });
  }
});

## Future Considerations

### Performance
- Template caching
- Optimized sync strategies
- Batch updates
- Debounce document updates
- Lazy loading of template content

### Security
- Field-level access control
- Input validation
- XSS prevention
- Document access permissions
- Audit logging

### Extensibility
- Custom field types
- Plugin system
- Template inheritance
- Custom event handlers
- Integration with other services

### User Experience
- Loading animations
- Save indicators
- Error feedback
- Undo/redo for linked updates
- Offline support

### Sam Integration Features
- Template suggestion
- Content validation
- Field auto-completion
- Context-aware assistance
- Document structure analysis

### Document Templates
- Version control
- Template categories
- Dynamic sections
- Conditional content
- Required fields validation

### Firestore Integration
- Real-time updates
- Batch processing
- Transaction handling
- Error recovery
- Change tracking

### Form Elements
- Custom input types
- Validation rules
- Dependent fields
- Calculated values
- Multi-select options

### API Considerations
- Event system
- Plugin architecture
- Custom renderers
- State management
- Error handling

## Implementation Priorities
1. Core editor functionality
2. Template system
3. Firestore integration
4. Task linking
5. Sam integration
6. Performance optimization
7. Security features
8. Extended functionality

## Notes
- All implementations should consider both web and mobile platforms
- Security should be implemented at both client and server level
- Performance monitoring should be implemented for all features
- Documentation should be maintained for all features
- Testing should cover all core functionality


## Templates with palceholders
Have document templates with placeholders and data attributes that map to Firestore fields:
<div class="work-order" data-remodl="workOrder" data-remodl-ref="workorders/{{id}}">
  <h1 data-remodl-field="title">{{title}}</h1>
  <div class="client-info" data-remodl-field="client">
    <p>Client: {{client.name}}</p>
    <p>Address: {{client.address}}</p>
  </div>
  <div class="scope" data-remodl-field="scope">{{scope}}</div>
  <!-- etc -->
</div>

2. Create a method to apply the template to a document:
```dart Future<String> applyTemplateToDocument({
  required String templateHtml,
  required DocumentReference workOrderRef,
}) async {
  try {
    final workOrderDoc = await workOrderRef.get();
    final workOrderData = workOrderDoc.data() as Map<String, dynamic>;
    
    // Replace placeholders with actual data
    String processedHtml = templateHtml;
    
    // Replace simple fields
    processedHtml = processedHtml.replaceAll('{{id}}', workOrderRef.id);
    processedHtml = processedHtml.replaceAll('{{title}}', workOrderData['title']);
    
    // Handle nested objects
    if (workOrderData['client'] != null) {
      processedHtml = processedHtml.replaceAll(
        '{{client.name}}', 
        workOrderData['client']['name']
      );
      processedHtml = processedHtml.replaceAll(
        '{{client.address}}', 
        workOrderData['client']['address']
      );
    }

    // Add data attributes for two-way sync
    processedHtml = processedHtml.replaceAll(
      'data-remodl-ref="workorders/{{id}}"',
      'data-remodl-ref="workorders/${workOrderRef.id}"'
    );

    return processedHtml;
  } catch (e) {
    print('Error applying template: $e');
    return templateHtml;
  }
}```

3. Add two-way synce to update Firestore when docuemnt changes


void _setupWorkOrderSync() {
  controller.callbacks.onChangeContent = (content) async {
    if (content == null) return;
    
    // Parse HTML to find work order reference
    final document = parse(content);
    final workOrderDiv = document.querySelector('[data-remodl="workOrder"]');
    if (workOrderDiv == null) return;
    
    final workOrderRef = workOrderDiv.getAttribute('data-remodl-ref');
    if (workOrderRef == null) return;

    // Find all fields that need syncing
    final syncFields = document.querySelectorAll('[data-remodl-field]');
    
    // Create update map
    Map<String, dynamic> updates = {};
    for (final field in syncFields) {
      final fieldName = field.getAttribute('data-remodl-field');
      if (fieldName != null) {
        updates[fieldName] = field.text;
      }
    }

    // Update Firestore
    try {
      await FirebaseFirestore.instance.doc(workOrderRef).update({
        ...updates,
        'last_modified': FieldValue.serverTimestamp(),
        'last_modified_by': FFAppState().currentUser?.id,
      });
    } catch (e) {
      print('Error syncing work order: $e');
    }
  };
}


4. Create a method to initialize a document from a template
Future<void> initializeFromTemplate({
  required String templateId,
  required DocumentReference workOrderRef,
}) async {
  try {
    // Get template
    final templateDoc = await FirebaseFirestore.instance
        .collection('documentTemplates')
        .doc(templateId)
        .get();
    
    if (!templateDoc.exists) return;
    
    final templateHtml = templateDoc.get('html') as String;
    
    // Apply work order data to template
    final processedHtml = await applyTemplateToDocument(
      templateHtml: templateHtml,
      workOrderRef: workOrderRef,
    );

    // Set editor content
    await controller.setText(processedHtml);
    
    // Set up sync
    _setupWorkOrderSync();
    
  } catch (e) {
    print('Error initializing from template: $e');
  }
}

This approach would:
Allow templates to be stored in Firestore
Map document fields to Firestore fields
Maintain two-way sync between the document and Firestore
Track changes and modifications
Allow Sam to focus on generating the Firestore data rather than the document HTML
Would you like me to:
Add more sophisticated template processing?
Add validation for required fields?
Add support for different types of templates?
Add error handling and recovery?

### Array Handling in Templates

Example Firestore document (/workorders/123):
{
  "title": "Kitchen Remodel",
  "materials": [
    { "id": "mat1", "name": "Cabinets", "quantity": 12, "unit": "pieces" },
    { "id": "mat2", "name": "Countertop", "quantity": 45, "unit": "sqft" }
  ],
  "tasks": [
    { "id": "task1", "title": "Demo", "status": "completed" },
    { "id": "task2", "title": "Cabinet Install", "status": "pending" }
  ]
}

HTML Template Implementation:
<div class="work-order" data-remodl="workOrder" data-remodl-ref="workorders/{{id}}">
  <h1 data-remodl-field="title">{{title}}</h1>
  
  <!-- Array Section: Materials -->
  <div data-remodl-array="materials">
    <!-- Template for each material item -->
    <div data-remodl-array-item="materials" data-remodl-item-template>
      <input type="text" data-remodl-array-field="name" placeholder="Material Name">
      <input type="number" data-remodl-array-field="quantity" placeholder="Quantity">
      <input type="text" data-remodl-array-field="unit" placeholder="Unit">
    </div>
    <!-- Add button for new items -->
    <button data-remodl-array-add="materials">Add Material</button>
  </div>

  <!-- Array Section: Tasks -->
  <div data-remodl-array="tasks">
    <!-- Template for each task item -->
    <div data-remodl-array-item="tasks" data-remodl-item-template>
      <input type="text" data-remodl-array-field="title" placeholder="Task Title">
      <input type="checkbox" 
             data-remodl-array-field="status"
             data-remodl-value-checked="completed"
             data-remodl-value-unchecked="pending">
    </div>
    <button data-remodl-array-add="tasks">Add Task</button>
  </div>
</div>

### JavaScript Handling in document.html
// Add array item handling
document.addEventListener('click', function(e) {
  // Handle add button clicks
  if (e.target.matches('[data-remodl-array-add]')) {
    const arrayName = e.target.getAttribute('data-remodl-array-add');
    const template = document.querySelector(`[data-remodl-array="${arrayName}"] [data-remodl-item-template]`);
    const newItem = template.cloneNode(true);
    newItem.removeAttribute('data-remodl-item-template');
    template.parentNode.insertBefore(newItem, e.target);
    
    postChannelMessage({
      'type': 'toDart: arrayItemAdded',
      'arrayName': arrayName
    });
  }
  
  // Handle remove button clicks
  if (e.target.matches('[data-remodl-array-remove]')) {
    const arrayItem = e.target.closest('[data-remodl-array-item]');
    const arrayName = arrayItem.getAttribute('data-remodl-array-item');
    arrayItem.remove();
    
    postChannelMessage({
      'type': 'toDart: arrayItemRemoved',
      'arrayName': arrayName
    });
  }
});

// Handle array item changes
document.addEventListener('change', function(e) {
  if (e.target.matches('[data-remodl-array-field]')) {
    const arrayItem = e.target.closest('[data-remodl-array-item]');
    const arrayName = arrayItem.getAttribute('data-remodl-array-item');
    const fieldName = e.target.getAttribute('data-remodl-array-field');
    const value = e.target.type === 'checkbox' ? e.target.checked : e.target.value;
    
    postChannelMessage({
      'type': 'toDart: arrayItemChanged',
      'arrayName': arrayName,
      'fieldName': fieldName,
      'value': value,
      'itemIndex': Array.from(arrayItem.parentNode.children).indexOf(arrayItem)
    });
  }
});

### Dart Implementation
void _handleArrayChanges(Map<String, dynamic> message) async {
  final docRef = widget.documentReference;
  if (docRef == null) return;

  try {
    final doc = await docRef.get();
    final data = doc.data() as Map<String, dynamic>;
    
    switch (message['type']) {
      case 'toDart: arrayItemAdded':
        final arrayName = message['arrayName'];
        final currentArray = List.from(data[arrayName] ?? []);
        currentArray.add({});  // Add empty item
        
        await docRef.update({
          arrayName: currentArray,
          'metadata.last_modified': FieldValue.serverTimestamp(),
          'metadata.last_modified_by': FFAppState().currentUser?.id,
        });
        break;
        
      case 'toDart: arrayItemRemoved':
        final arrayName = message['arrayName'];
        final currentArray = List.from(data[arrayName] ?? []);
        currentArray.removeAt(message['itemIndex']);
        
        await docRef.update({
          arrayName: currentArray,
          'metadata.last_modified': FieldValue.serverTimestamp(),
          'metadata.last_modified_by': FFAppState().currentUser?.id,
        });
        break;
        
      case 'toDart: arrayItemChanged':
        final arrayName = message['arrayName'];
        final fieldName = message['fieldName'];
        final value = message['value'];
        final itemIndex = message['itemIndex'];
        
        final currentArray = List.from(data[arrayName] ?? []);
        if (itemIndex < currentArray.length) {
          currentArray[itemIndex][fieldName] = value;
          
          await docRef.update({
            arrayName: currentArray,
            'metadata.last_modified': FieldValue.serverTimestamp(),
            'metadata.last_modified_by': FFAppState().currentUser?.id,
          });
        }
        break;
    }
  } catch (e) {
    print('Error handling array change: $e');
  }
}

### Template Processing for Arrays


String _processArrayTemplate(String templateHtml, Map<String, dynamic> data) {
  final document = parse(templateHtml);
  
  // Process each array section
  document.querySelectorAll('[data-remodl-array]').forEach((arraySection) {
    final arrayName = arraySection.getAttribute('data-remodl-array');
    final template = arraySection.querySelector('[data-remodl-item-template]');
    final array = data[arrayName] as List?;
    
    if (array != null && template != null) {
      // Remove existing items (except template)
      arraySection.children
          .where((e) => !e.hasAttribute('data-remodl-item-template'))
          .forEach((e) => e.remove());
      
      // Add items from data
      for (final item in array) {
        final newItem = template.clone(true);
        newItem.removeAttribute('data-remodl-item-template');
        
        // Fill in field values
        newItem.querySelectorAll('[data-remodl-array-field]').forEach((field) {
          final fieldName = field.getAttribute('data-remodl-array-field');
          if (item[fieldName] != null) {
            if (field.tagName.toLowerCase() == 'input') {
              if (field.getAttribute('type') == 'checkbox') {
                field.setAttribute('checked', item[fieldName] == true);
              } else {
                field.setAttribute('value', item[fieldName].toString());
              }
            } else {
              field.text = item[fieldName].toString();
            }
          }
        });
        
        arraySection.insertBefore(newItem, template);
      }
    }
  });
  
  return document.outerHtml;
}