---
name: admin-portal
description: Admin portal conventions (Thymeleaf templates, ModelVisitor controllers, form patterns). Use when creating or modifying client admin views.
---

# Admin Portal

## Overview

The admin portal is an OAuth2-protected web UI where clients manage their content. Each client gets a Thymeleaf template and a controller implementing `ModelVisitor`. The `IndexController` dispatches to the correct view based on the authenticated user's email domain.

## Dispatch Mechanism

`IndexController` extracts the domain from the user's email via regex (`(?<=@)[^.]+(?=\\.)`), prepends `clients/`, and finds the matching `ModelVisitor`:

```
info@come-in-and-find-out.ch  →  domain: come-in-and-find-out  →  view: clients/come-in-and-find-out
info@gaiapeeps.com            →  domain: gaiapeeps             →  view: clients/gaiapeeps
```

If no `ModelVisitor` returns a matching view name, `AccessDeniedException` is thrown.

## ModelVisitor Interface

```java
public interface ModelVisitor {
    ModelAndView getModel();
}
```

Every client admin controller implements this. The `getModel()` method returns a `ModelAndView` with:
- View name: `clients/<domain-prefix>`
- Model objects: the items to display in the form

## Controller Variants

### Simple (no images, no cache)

```java
@Controller
@RequiredArgsConstructor
@Slf4j
class <Name>Controller implements ModelVisitor {
    private final <Name>Converter <name>Converter;

    private final <Name>Repository <name>Repository;

    @PostMapping("/<site-name>")
    String submit(@RequestParam Map<String, String> params) {
        log.info("Upload processor starts: <site-name>");
        List<<Name>Entity> items = <name>Converter.processItems(params);
        <name>Repository.saveAll(items);
        <name>Repository.deleteAllById(<name>Converter.filterIdsToDelete(items));
        return "redirect:/";
    }

    @Override
    public ModelAndView getModel() {
        ModelAndView model = new ModelAndView("clients/<domain-prefix>");
        model.addObject("items", <name>Repository.findAll());
        return model;
    }
}
```

### Complex (images, cache, categories)

```java
@Controller
@RequiredArgsConstructor
@Slf4j
class <Name>Controller implements ModelVisitor {
    private final <Name>Converter <name>Converter;

    private final <Name>Repository <name>Repository;

    @CacheEvict(value = "<name>Items", allEntries = true)
    @PostMapping("/<site-name>")
    String submit(@RequestParam String category, @RequestParam Map<String, String> params, @RequestParam Map<String, MultipartFile> files) {
        log.info("Upload processor starts: <site-name>");
        List<<Name>Entity> items = <name>Converter.processItems(category, params, files);
        <name>Repository.saveAll(<name>Converter.filterItemsToInsert(items));
        <name>Converter.filterItemsToUpdate(items).forEach(<name>Repository::update);
        <name>Repository.deleteAllById(<name>Converter.filterIdsToDelete(items));
        return "redirect:/";
    }

    @Override
    public ModelAndView getModel() {
        ModelAndView model = new ModelAndView("clients/<domain-prefix>");
        List<String> categories = List.of("cat1", "cat2");
        Map<String, List<<Name>Entity>> items = categories
                .stream()
                .collect(Collectors.toMap(Function.identity(), <name>Repository::get));
        model.addObject("categories", categories);
        model.addObject("items", items);
        return model;
    }
}
```

## Template Structure

All templates use:
- Bootstrap 5 CSS + Icons (CDN)
- `xmlns:th="http://www.thymeleaf.org"`
- Title image (`/img/clients/<name>.webp`)
- Header bar with: Save button, New item button, Grafana link, ntfy link, Logout button
- `items[n].field` form naming convention

### Template File Location

```
src/main/resources/templates/clients/<domain-prefix>.html
```

### Simple Template (table layout)

Used when items have no images — just text fields in a table.

```html
<form th:action="@{'/site-name'}" method="post">
    <table class="table table-bordered">
        <thead>
        <tr>
            <th>Field1</th>
            <th>Field2</th>
            <th>Actions</th>
        </tr>
        </thead>
        <tbody id="tableBodyItems">
        <tr th:each="item, iter: ${items}">
            <td>
                <textarea class="form-control" th:name="|items[${iter.index}].field1|" rows="3" th:text="${item.field1}"></textarea>
            </td>
            <td>
                <textarea class="form-control" th:name="|items[${iter.index}].field2|" rows="3" th:text="${item.field2}"></textarea>
            </td>
            <td style="text-align: center;">
                <input type="checkbox" class="btn-check" th:name="|items[${iter.index}].delete|" th:id="'btn-check-' + ${iter.index}">
                <label class="btn btn-outline-danger" th:for="'btn-check-' + ${iter.index}"><i class="bi bi-trash"></i></label>
                <input type="hidden" th:name="|items[${iter.index}].id|" th:value="${item.id}">
            </td>
        </tr>
        </tbody>
    </table>
</form>
```

### Complex Template (accordion + images)

Used when items are grouped by category and have image uploads.

- `enctype="multipart/form-data"` on the form
- Category passed as hidden or query param: `th:action="@{'/site-name?category=' + ${category}}"`
- Accordion groups items by category
- Image fields: thumbnail preview + file input (hidden, triggered by button)
- File validation in JS: JPEG/PNG/GIF only, max 4MB

## Form Data Convention

- Fields: `items[0].title`, `items[0].link`, `items[1].title`, etc.
- Delete checkbox: `items[n].delete` (value `"on"` when checked)
- Hidden ID: `items[n].id` (preserves existing item identity)
- File uploads: `items[n].image`, `items[n].thumbnail` as `MultipartFile`

## JavaScript Patterns

### addRow()

Dynamically adds a new row to the table/accordion. Calculates the next index from existing row count:

```javascript
function addRow() {
    const tableBody = document.getElementById('tableBodyItems');
    const tableSize = tableBody.querySelectorAll('tr').length;
    const newRow = document.createElement('tr');
    newRow.innerHTML = `
        <td><textarea class="form-control" name="items[${tableSize}].field1" rows="3"></textarea></td>
        ...
    `;
    tableBody.insertBefore(newRow, tableBody.firstChild);
}
```

New rows are inserted at the top (before `firstChild`).

### Submit spinner

All submit buttons disable themselves and show a spinner on click:

```javascript
document.querySelectorAll('button[type="submit"]').forEach(button => {
    button.addEventListener('click', function () {
        button.disabled = true;
        button.querySelector('.spinner-border').classList.remove('d-none');
        setTimeout(() => { button.closest('form').submit(); }, 50);
    });
});
```

### File upload (image templates only)

- `selectFile(button)` — triggers the hidden file input
- `updateButtonText(input)` — validates file type/size, updates preview thumbnail via `FileReader`
- `showWarningModal(message)` — Bootstrap modal for validation errors

## Header Bar Template

Shared across all client templates:

```html
<div class="mb-3 d-flex align-items-center">
    <div>
        <button class="btn btn-success me-2" type="submit">
            <i class="bi bi-floppy me-1"></i> Save changes
            <span class="spinner-border spinner-border-sm d-none ms-2" role="status"></span>
        </button>
        <button class="btn btn-outline-primary" type="button" onclick="addRow()">
            <i class="bi bi-plus-lg"></i> New item
        </button>
    </div>
    <div class="ms-auto d-flex align-items-center gap-3">
        <a th:href="@{'/grafana'}" target="_blank"><img src="/img/grafana.svg" width="32" alt=""></a>
        <a th:href="@{'/ntfy'}" target="_blank"><img src="/img/ntfy.svg" width="32" alt=""></a>
        <a th:href="@{'/logout'}" class="btn btn-outline-secondary">
            <i class="bi bi-box-arrow-right me-1"></i> Logout
        </a>
    </div>
</div>
```

## Checklist

When creating a new admin portal view:

1. Create template at `templates/clients/<domain-prefix>.html`
2. Create controller implementing `ModelVisitor` in `controllers/clients/<name>/`
3. Ensure the view name in `ModelAndView` matches `clients/<domain-prefix>`
4. Add title image at `src/main/resources/static/img/clients/<name>.webp`
5. The `IndexController` auto-discovers the new `ModelVisitor` via Spring's `List<ModelVisitor>` injection
