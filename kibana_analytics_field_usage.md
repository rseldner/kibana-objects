
# Kibana Analytics Field Usage - Runtime Fields

Data view targeting `.kibana_analytics`, providing runtime fields that extract
Elasticsearch field references from saved Kibana objects for usage analysis and
mapping optimization reporting.

## Runtime Fields

| Field Name | Type | Applies To | Source Path | Method | Notes |
|---|---|---|---|---|---|
| `field_usage_controls` | keyword | `dashboard` | `dashboard.controlGroupInput.panelsJSON` → `fieldName` | String scan | Fields used as interactive filter controls (options list, range slider)|
| `field_usage_panels` | keyword | `dashboard` | `dashboard.panelsJSON` → `sourceField` | String scan | Fields used in embedded Lens panels within dashboards |
| `field_usage_lens` | keyword | `lens` | `lens.state.datasourceStates.formBased.layers[*].columns[*].sourceField` | Object traversal | Fields used in standalone Lens visualizations|
| `field_usage_search` | keyword | `search` | `search.columns[*]` | Array iteration | Fields selected as visible columns in saved Discover searches|
| `field_usage_maps` | keyword | `map` | `map.layerListJSON` → `field`, `term`, `geoField` | String scan | Fields used in Maps layers across data, join, and geo layer types |
| `field_usage_vis` | keyword | `visualization` | `visualization.visState` → `aggs[*].params.field` | String scan scoped to aggs | Fields used in legacy aggregation-based visualizations|
| `field_usage_graph` | keyword | `graph-workspace` | `graph-workspace.wsState` → `selectedFields[*].name` | String scan (double-encoded) | Fields configured for graph exploration|
## Excluded Object Types

| Type | Reason |
|---|---|
| `canvas-workpad` | Canvas expression language is freeform and not reliably parseable for field references.  It is also deprecated. |
| `canvas-workpad-template` | Same as above; templates typically use built-in `demodata` rather than real ES fields |
| `visualization` (vega/vega_map) | Vega spec uses non-standard JSON syntax with comments and unquoted keys |

## Usage Notes

- All scripts include a **type guard** as the first statement, returning immediately for non-matching document types, minimizing compute overhead across the full index.
- All scripts deduplicate emitted values using a `HashSet` and enforce a **maximum of 100 emitted values** per document.

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
    "timeFieldName": "created_at",
    "runtimeFieldMap": {
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
