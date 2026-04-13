
# Kibana Analytics Field Usage — Runtime Fields

Data view targeting `.kibana_analytics`, providing runtime fields that extract,
normalize, and categorize Elasticsearch field references across all major saved
object types — enabling field usage analysis, impact assessment, and mapping
optimization reporting directly in Kibana visualizations.

## Native Fields of Interest

| Field Name | Type | Notes |
|---|---|---|
| `type` | keyword | Kibana object type — `dashboard`, `lens`, `search`, `map`, `visualization`, `graph-workspace`, etc. |
| `namespaces` | keyword | Kibana space(s) the object belongs to. |

## Runtime Fields

| Field Name | Type | Applies To | Source Path | Method | Notes |
|---|---|---|---|---|---|
| `field_usage_object_title` | keyword | all | Type-specific title field per object type | Conditional assignment | Normalized title across all object types. Supports `dashboard`, `lens`, `search`, `visualization`, `map`, `graph-workspace`, `index-pattern`. |
| `field_usage_index_patterns` | keyword | all | `references[*]` where `type = index-pattern` → `id` | Array iteration | Index pattern IDs referenced by the object. Falls back to `graph-workspace.legacyIndexPatternRef` for graph workspaces. |
| `field_usage_all` | keyword | all | All type-specific sources combined | Mixed | All ES field references from the object, deduplicated across all sources via shared HashSet. Combines logic of all type-specific fields below. |
| `field_usage_controls` | keyword | `dashboard` | `dashboard.controlGroupInput.panelsJSON` → `fieldName` | String scan | Fields used as interactive filter controls (options list, range slider). Supports compact and spaced JSON variants. |
| `field_usage_panels` | keyword | `dashboard` | `dashboard.panelsJSON` → `sourceField` | String scan | Fields used in embedded Lens panels within dashboards. Excludes `___records___` and `@timestamp`. Supports compact and spaced JSON variants. |
| `field_usage_lens` | keyword | `lens` | `lens.state.datasourceStates.formBased.layers[*].columns[*].sourceField` | Object traversal | Fields used in standalone Lens visualizations. Excludes `___records___` and `@timestamp`. |
| `field_usage_search` | keyword | `search` | `search.columns[*]` | Array iteration | Fields selected as visible columns in saved Discover searches. Excludes `___records___` and `@timestamp`. |
| `field_usage_maps` | keyword | `map` | `map.layerListJSON` → `field`, `term`, `geoField` | String scan | Fields used in Maps layers across data, join, and geo layer types. Supports compact and spaced JSON variants. |
| `field_usage_vis` | keyword | `visualization` | `visualization.visState` → `aggs[*].params.field` | String scan scoped to aggs | Fields used in legacy aggregation-based visualizations. Vega and Vega Map types are skipped as their specs are not reliably parseable. |
| `field_usage_graph` | keyword | `graph-workspace` | `graph-workspace.wsState` → `selectedFields[*].name` | String scan (double-encoded) | Fields configured for graph exploration. `wsState` is double-encoded JSON requiring escaped quote matching. Heuristic filter excludes icon classes and color values. |

## Excluded Object Types

| Type | Reason |
|---|---|
| `canvas-workpad` | Canvas expression language is freeform and not reliably parseable for field references. |
| `canvas-workpad-template` | Same as above; templates typically use built-in `demodata` rather than real ES fields. |
| `visualization` (vega/vega_map) | Vega spec uses non-standard JSON syntax with comments and unquoted keys. |

## General Notes

- All scripts include a **type guard** as the first statement, returning immediately for non-matching document types to minimize compute overhead.
- All scripts deduplicate emitted values using a `HashSet` and enforce a **maximum of 100 emitted values** per document.
- `field_usage_all` shares a single `HashSet` across all extraction sources within a document, ensuring true cross-source deduplication.
- `field_usage_index_patterns` emits index pattern **IDs** (UUIDs) for most types. Cross-reference against `index-pattern` saved objects to resolve to human-readable titles.

<details>
<summary>
Create Data View w/ needed runtime mappings
</summary>

```
POST kbn:api/data_views/data_view
{
  "data_view": {
    "id": ".kibana_analytics_field_usage",
    "title": ".kibana_analytics",
    "name": "Kibana Analytics Field Usage",
    "timeFieldName": "updated_at",
    "runtimeFieldMap": {
      "field_usage_object_title": {
        "type": "keyword",
        "script": {
          "source": """
def type = params._source?.type;
if (type == null) return;

def title = null;

if ('dashboard'.equals(type))              title = params._source?.dashboard?.title;
else if ('lens'.equals(type))              title = params._source?.lens?.title;
else if ('search'.equals(type))            title = params._source?.search?.title;
else if ('visualization'.equals(type))     title = params._source?.visualization?.title;
else if ('map'.equals(type))               title = params._source?.map?.title;
else if ('graph-workspace'.equals(type)) {
  def gw = params._source?.get('graph-workspace');
  if (gw != null) title = gw?.title;
}
else if ('index-pattern'.equals(type)) {
  def ip = params._source?.get('index-pattern');
  if (ip != null) title = ip?.title;
}

if (title != null && title.length() > 0) emit(title);"""
        }
      },
      "field_usage_index_patterns": {
        "type": "keyword",
        "script": {
          "source": """
def seen = new HashSet();
def refs = params._source?.references;

if (refs != null) {
  for (def ref : refs) {
    if ('index-pattern'.equals(ref?.type)) {
      def val = ref?.id;
      if (val != null && val.length() > 0 && seen.add(val) && seen.size() < 100) {
        emit(val);
      }
    }
  }
}

def gw = params._source?.get('graph-workspace');
if (gw != null) {
  def legacyRef = gw?.legacyIndexPatternRef;
  if (legacyRef != null && legacyRef.length() > 0 && seen.add(legacyRef)) {
    emit(legacyRef);
  }
}"""
        }
      },
      "field_usage_all": {
        "type": "keyword",
        "script": {
          "source": """
def seen = new HashSet();
def type = params._source?.type;
if (type == null) return;

if ('dashboard'.equals(type)) {
  def controlsJSON = params._source?.dashboard?.controlGroupInput?.panelsJSON;
  if (controlsJSON != null) {
    def patterns = new ArrayList();
    patterns.add('"fieldName":"');
    patterns.add('"fieldName": "');
    for (def pattern : patterns) {
      def start = 0;
      while ((start = controlsJSON.indexOf(pattern, start)) != -1) {
        start += pattern.length();
        def end = controlsJSON.indexOf('"', start);
        if (end == -1) break;
        def val = controlsJSON.substring(start, end);
        if (val.length() > 0 && seen.add(val) && seen.size() < 100) emit(val);
        start = end + 1;
      }
    }
  }

  def panelsJSON = params._source?.dashboard?.panelsJSON;
  if (panelsJSON != null) {
    def patterns = new ArrayList();
    patterns.add('"sourceField":"');
    patterns.add('"sourceField": "');
    for (def pattern : patterns) {
      def start = 0;
      while ((start = panelsJSON.indexOf(pattern, start)) != -1) {
        start += pattern.length();
        def end = panelsJSON.indexOf('"', start);
        if (end == -1) break;
        def val = panelsJSON.substring(start, end);
        if (val.length() > 0 && !val.equals('___records___') && !val.equals('@timestamp') && seen.add(val) && seen.size() < 100) emit(val);
        start = end + 1;
      }
    }
  }
}

else if ('lens'.equals(type)) {
  def layers = params._source?.lens?.state?.datasourceStates?.formBased?.layers;
  if (layers != null) {
    for (def layer : layers.values()) {
      def cols = layer?.columns;
      if (cols == null) continue;
      for (def col : cols.values()) {
        def val = col?.sourceField;
        if (val != null && val.length() > 0
            && !val.equals('___records___')
            && !val.equals('@timestamp')
            && seen.add(val)
            && seen.size() < 100) {
          emit(val);
        }
      }
    }
  }
}

else if ('search'.equals(type)) {
  def cols = params._source?.search?.columns;
  if (cols != null) {
    for (def val : cols) {
      if (val != null && val.length() > 0
          && !val.equals('___records___')
          && !val.equals('@timestamp')
          && seen.add(val)
          && seen.size() < 100) {
        emit(val);
      }
    }
  }
}

else if ('map'.equals(type)) {
  def layerStr = params._source?.map?.layerListJSON;
  if (layerStr != null) {
    def patterns = new ArrayList();
    patterns.add('"field":"');
    patterns.add('"field": "');
    patterns.add('"term":"');
    patterns.add('"term": "');
    patterns.add('"geoField":"');
    patterns.add('"geoField": "');
    for (def pattern : patterns) {
      def start = 0;
      while ((start = layerStr.indexOf(pattern, start)) != -1) {
        start += pattern.length();
        def end = layerStr.indexOf('"', start);
        if (end == -1) break;
        def val = layerStr.substring(start, end);
        if (val.length() > 0
            && !val.equals('___records___')
            && !val.equals('@timestamp')
            && seen.add(val)
            && seen.size() < 100) {
          emit(val);
        }
        start = end + 1;
      }
    }
  }
}

else if ('visualization'.equals(type)) {
  def visStateStr = params._source?.visualization?.visState;
  if (visStateStr != null
      && !visStateStr.contains('"type":"vega"')
      && !visStateStr.contains('"type": "vega"')
      && !visStateStr.contains('"type":"vega_map"')
      && !visStateStr.contains('"type": "vega_map"')) {
    def aggsStart = visStateStr.indexOf('"aggs":[');
    if (aggsStart == -1) aggsStart = visStateStr.indexOf('"aggs": [');
    if (aggsStart != -1) {
      def aggsStr = visStateStr.substring(aggsStart);
      def patterns = new ArrayList();
      patterns.add('"field":"');
      patterns.add('"field": "');
      for (def pattern : patterns) {
        def start = 0;
        while ((start = aggsStr.indexOf(pattern, start)) != -1) {
          start += pattern.length();
          def end = aggsStr.indexOf('"', start);
          if (end == -1) break;
          def val = aggsStr.substring(start, end);
          if (val.length() > 0
              && !val.equals('___records___')
              && !val.equals('@timestamp')
              && seen.add(val)
              && seen.size() < 100) {
            emit(val);
          }
          start = end + 1;
        }
      }
    }
  }
}

else if ('graph-workspace'.equals(type)) {
  def gw = params._source?.get('graph-workspace');
  if (gw != null) {
    def wsState = gw?.wsState;
    if (wsState != null) {
      def patterns = new ArrayList();
      patterns.add('\\"name\\":\\"');
      patterns.add('\\"name\\": \\"');
      for (def pattern : patterns) {
        def start = 0;
        while ((start = wsState.indexOf(pattern, start)) != -1) {
          start += pattern.length();
          def end = wsState.indexOf('\\"', start);
          if (end == -1) break;
          def val = wsState.substring(start, end);
          if (val.length() > 0 && !val.contains(' ')
              && !val.startsWith('fa-') && !val.startsWith('#')
              && seen.add(val) && seen.size() < 100) {
            emit(val);
          }
          start = end + 1;
        }
      }
    }
  }
}"""
        }
      },
      "field_usage_controls": {
        "type": "keyword",
        "script": {
          "source": """
if (!'dashboard'.equals(params._source?.type)) return;
def seen = new HashSet();
def start = 0;
def end = 0;
def val = '';
def controlsJSON = params._source?.dashboard?.controlGroupInput?.panelsJSON;
if (controlsJSON == null) return;

def patterns = new ArrayList();
patterns.add('"fieldName":"');
patterns.add('"fieldName": "');
for (def pattern : patterns) {
  start = 0;
  while ((start = controlsJSON.indexOf(pattern, start)) != -1) {
    start += pattern.length();
    end = controlsJSON.indexOf('"', start);
    if (end == -1) break;
    val = controlsJSON.substring(start, end);
    if (val.length() > 0 && seen.add(val) && seen.size() < 100) emit(val);
    start = end + 1;
  }
}"""
        }
      },
      "field_usage_panels": {
        "type": "keyword",
        "script": {
          "source": """
if (!'dashboard'.equals(params._source?.type)) return;
def seen = new HashSet();
def start = 0;
def end = 0;
def val = '';
def panelsJSON = params._source?.dashboard?.panelsJSON;
if (panelsJSON == null) return;

def patterns = new ArrayList();
patterns.add('"sourceField":"');
patterns.add('"sourceField": "');
for (def pattern : patterns) {
  start = 0;
  while ((start = panelsJSON.indexOf(pattern, start)) != -1) {
    start += pattern.length();
    end = panelsJSON.indexOf('"', start);
    if (end == -1) break;
    val = panelsJSON.substring(start, end);
    if (val.length() > 0 && !val.equals('___records___') && !val.equals('@timestamp') && seen.add(val) && seen.size() < 100) emit(val);
    start = end + 1;
  }
}"""
        }
      },
      "field_usage_lens": {
        "type": "keyword",
        "script": {
          "source": """
if (!'lens'.equals(params._source?.type)) return;
def seen = new HashSet();
def layers = params._source?.lens?.state?.datasourceStates?.formBased?.layers;
if (layers == null) return;

for (def layer : layers.values()) {
  def cols = layer?.columns;
  if (cols == null) continue;
  for (def col : cols.values()) {
    def val = col?.sourceField;
    if (val != null && val.length() > 0
        && !val.equals('___records___')
        && !val.equals('@timestamp')
        && seen.add(val)
        && seen.size() < 100) {
      emit(val);
    }
  }
}"""
        }
      },
      "field_usage_search": {
        "type": "keyword",
        "script": {
          "source": """
if (!'search'.equals(params._source?.type)) return;
def seen = new HashSet();
def cols = params._source?.search?.columns;
if (cols == null) return;

for (def val : cols) {
  if (val != null && val.length() > 0
      && !val.equals('___records___')
      && !val.equals('@timestamp')
      && seen.add(val)
      && seen.size() < 100) {
    emit(val);
  }
}"""
        }
      },
      "field_usage_maps": {
        "type": "keyword",
        "script": {
          "source": """
if (!'map'.equals(params._source?.type)) return;
def seen = new HashSet();
def layerStr = params._source?.map?.layerListJSON;
if (layerStr == null) return;

def patterns = new ArrayList();
patterns.add('"field":"');
patterns.add('"field": "');
patterns.add('"term":"');
patterns.add('"term": "');
patterns.add('"geoField":"');
patterns.add('"geoField": "');
for (def pattern : patterns) {
  def start = 0;
  while ((start = layerStr.indexOf(pattern, start)) != -1) {
    start += pattern.length();
    def end = layerStr.indexOf('"', start);
    if (end == -1) break;
    def val = layerStr.substring(start, end);
    if (val.length() > 0
        && !val.equals('___records___')
        && !val.equals('@timestamp')
        && seen.add(val)
        && seen.size() < 100) {
      emit(val);
    }
    start = end + 1;
  }
}"""
        }
      },
      "field_usage_vis": {
        "type": "keyword",
        "script": {
          "source": """
if (!'visualization'.equals(params._source?.type)) return;
def seen = new HashSet();
def visStateStr = params._source?.visualization?.visState;
if (visStateStr == null) return;

if (visStateStr.contains('"type":"vega"') || visStateStr.contains('"type": "vega"')
 || visStateStr.contains('"type":"vega_map"') || visStateStr.contains('"type": "vega_map"')) return;

def aggsStart = visStateStr.indexOf('"aggs":[');
if (aggsStart == -1) aggsStart = visStateStr.indexOf('"aggs": [');
if (aggsStart == -1) return;
def aggsStr = visStateStr.substring(aggsStart);

def patterns = new ArrayList();
patterns.add('"field":"');
patterns.add('"field": "');
for (def pattern : patterns) {
  def start = 0;
  while ((start = aggsStr.indexOf(pattern, start)) != -1) {
    start += pattern.length();
    def end = aggsStr.indexOf('"', start);
    if (end == -1) break;
    def val = aggsStr.substring(start, end);
    if (val.length() > 0
        && !val.equals('___records___')
        && !val.equals('@timestamp')
        && seen.add(val)
        && seen.size() < 100) {
      emit(val);
    }
    start = end + 1;
  }
}"""
        }
      },
      "field_usage_graph": {
        "type": "keyword",
        "script": {
          "source": """
if (!'graph-workspace'.equals(params._source?.type)) return;
def seen = new HashSet();
def gw = params._source?.get('graph-workspace');
if (gw == null) return;
def wsState = gw?.wsState;
if (wsState == null) return;

def patterns = new ArrayList();
patterns.add('\\"name\\":\\"');
patterns.add('\\"name\\": \\"');
for (def pattern : patterns) {
  def start = 0;
  while ((start = wsState.indexOf(pattern, start)) != -1) {
    start += pattern.length();
    def end = wsState.indexOf('\\"', start);
    if (end == -1) break;
    def val = wsState.substring(start, end);
    if (val.length() > 0 && !val.contains(' ')
        && !val.startsWith('fa-') && !val.startsWith('#')
        && seen.add(val) && seen.size() < 100) {
      emit(val);
    }
    start = end + 1;
  }
}"""
        }
      }
    }
  }
}
```


</details>
