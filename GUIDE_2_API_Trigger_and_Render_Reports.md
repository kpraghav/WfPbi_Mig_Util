# Triggering & Rendering Published Reports via API

## Complete Step-by-Step Guide

This guide covers how to call your published reports from an API,
pass parameters (input conditions), and render the output in your
application's UI — after the reports are live on Power BI Report Server.


## How Power BI Report Server API Works

Power BI Report Server exposes two APIs:

1. **Report Server REST API (v2.0)**
   - Base URL: `https://your-server/reports/api/v2.0`
   - Used for: listing reports, managing folders, getting metadata

2. **Report Execution URL**
   - Base URL: `https://your-server/reportserver`
   - Used for: rendering reports with parameters, exporting to PDF/Excel/HTML
   - This is the one you'll use most for your use case

Both use Windows Authentication (NTLM/Kerberos) by default.


## Prerequisites

1. Reports are published and working in the web portal (Guide 1 complete)
2. You know your Report Server URLs:
   - Web Portal: `https://your-server/reports`
   - Report Execution: `https://your-server/reportserver`
3. You have a service account with read/execute permissions on the reports
4. Your application backend can make HTTP requests to the Report Server


## Architecture Overview

```
User's Browser
    │
    │  1. User selects filters (year=2024, region=West)
    │     and clicks "Generate Report"
    ▼
Your Application Backend (Java / Node / Python / .NET)
    │
    │  2. Builds the Report Server URL with parameters
    │  3. Calls Report Server with service account credentials
    ▼
Power BI Report Server
    │
    │  4. Executes the report with given parameters
    │  5. Renders to requested format (PDF/HTML/Excel)
    ▼
Your Application Backend
    │
    │  6. Receives the rendered output
    │  7. Returns it to the browser
    ▼
User's Browser
    │
    │  8. Displays embedded HTML or downloads PDF/Excel
    ▼
User sees the report
```


## Method 1: Direct URL Rendering (Simplest)

The simplest way to show a report is to construct a URL that the
browser opens directly (or your app embeds in an iframe).


### URL Format

```
https://your-server/reportserver?/FolderPath/ReportName&PARAM1=value1&PARAM2=value2&rs:Format=FORMAT
```

### URL Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| Report path | After the `?`, use `/` separated path | `?/Sales/Quarterly_Summary` |
| Report params | Use `&ParamName=Value` | `&FISCAL_YEAR=2024` |
| `rs:Format` | Output format | `HTML5`, `PDF`, `EXCELOPENXML`, `CSV`, `IMAGE` |
| `rs:Command` | Command type | `Render` (default) |
| `rc:Toolbar` | Show/hide toolbar | `true` or `false` |
| `rc:Parameters` | Show/hide param bar | `Collapsed`, `true`, `false` |


### Example URLs

Render as interactive HTML (with toolbar):
```
https://your-server/reportserver?/Migrated Reports/Sales/Quarterly_Summary
    &FISCAL_YEAR=2024
    &REGION=West
    &rs:Format=HTML5
    &rc:Toolbar=true
```

Render as PDF (direct download):
```
https://your-server/reportserver?/Migrated Reports/Sales/Quarterly_Summary
    &FISCAL_YEAR=2024
    &REGION=West
    &rs:Format=PDF
```

Render as Excel:
```
https://your-server/reportserver?/Migrated Reports/Sales/Quarterly_Summary
    &FISCAL_YEAR=2024
    &rs:Format=EXCELOPENXML
```

Embed in iframe (hide toolbar, collapse parameters):
```
https://your-server/reportserver?/Migrated Reports/Sales/Quarterly_Summary
    &FISCAL_YEAR=2024
    &rs:Format=HTML5
    &rc:Toolbar=false
    &rc:Parameters=false
```


### Embed in Your UI (iframe)

The simplest UI integration is an iframe:

```html
<iframe
  src="https://your-server/reportserver?/Migrated Reports/Sales/Quarterly_Summary&FISCAL_YEAR=2024&rs:Format=HTML5&rc:Toolbar=false&rc:Parameters=false"
  width="100%"
  height="800px"
  frameborder="0">
</iframe>
```

Your frontend builds this URL dynamically based on user input:

```javascript
function renderReport(year, region) {
  const baseUrl = 'https://your-server/reportserver';
  const reportPath = '/Migrated Reports/Sales/Quarterly_Summary';
  const params = new URLSearchParams({
    'FISCAL_YEAR': year,
    'REGION': region,
    'rs:Format': 'HTML5',
    'rc:Toolbar': 'false',
    'rc:Parameters': 'false'
  });

  const iframe = document.getElementById('report-frame');
  iframe.src = `${baseUrl}?${reportPath}&${params.toString()}`;
}
```

Note: This requires the user's browser to have network access to the
Report Server AND the user must be authenticated (Windows Auth).
This works well for intranet apps where users are already on the domain.


## Method 2: Backend API Proxy (Recommended for Custom UIs)

For a proper API-driven flow where your backend controls access,
passes credentials, and returns the rendered report to the frontend.


### Java (Spring Boot) Implementation

Add this to your application backend:

```java
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;
import java.util.*;

@RestController
@RequestMapping("/api/reports")
public class ReportController {

    // Configure these for your environment
    private static final String REPORT_SERVER_URL = "https://your-server/reportserver";
    private static final String SERVICE_ACCOUNT_USER = "DOMAIN\\service_account";
    private static final String SERVICE_ACCOUNT_PASS = "password";

    /**
     * Render a report as PDF and return it to the frontend.
     *
     * Example call:
     *   GET /api/reports/render?path=/Sales/Quarterly_Summary
     *       &format=PDF&FISCAL_YEAR=2024&REGION=West
     */
    @GetMapping("/render")
    public ResponseEntity<byte[]> renderReport(
            @RequestParam String path,
            @RequestParam(defaultValue = "PDF") String format,
            @RequestParam Map<String, String> allParams) {

        // Build the Report Server URL
        StringBuilder url = new StringBuilder(REPORT_SERVER_URL);
        url.append("?").append(path);

        // Add report parameters (exclude our control params)
        Set<String> controlParams = Set.of("path", "format");
        allParams.forEach((key, value) -> {
            if (!controlParams.contains(key)) {
                url.append("&").append(key).append("=").append(value);
            }
        });

        // Set output format
        url.append("&rs:Format=").append(format);
        url.append("&rs:Command=Render");

        // Create a RestTemplate with NTLM authentication
        RestTemplate restTemplate = createNtlmRestTemplate();

        // Call Report Server
        ResponseEntity<byte[]> response = restTemplate.exchange(
            url.toString(),
            HttpMethod.GET,
            null,
            byte[].class
        );

        // Determine content type based on format
        MediaType contentType = switch (format.toUpperCase()) {
            case "PDF" -> MediaType.APPLICATION_PDF;
            case "EXCELOPENXML" -> MediaType.parseMediaType(
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
            case "CSV" -> MediaType.parseMediaType("text/csv");
            case "HTML5", "HTML4.0", "MHTML" -> MediaType.TEXT_HTML;
            case "IMAGE" -> MediaType.IMAGE_PNG;
            default -> MediaType.APPLICATION_OCTET_STREAM;
        };

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(contentType);

        // For downloadable formats, set filename
        if (Set.of("PDF", "EXCELOPENXML", "CSV").contains(format.toUpperCase())) {
            String filename = path.substring(path.lastIndexOf('/') + 1);
            String ext = switch (format.toUpperCase()) {
                case "PDF" -> ".pdf";
                case "EXCELOPENXML" -> ".xlsx";
                case "CSV" -> ".csv";
                default -> "";
            };
            headers.setContentDisposition(
                ContentDisposition.attachment().filename(filename + ext).build());
        }

        return ResponseEntity.ok()
            .headers(headers)
            .body(response.getBody());
    }

    /**
     * Render a report as embeddable HTML.
     */
    @GetMapping("/embed")
    public ResponseEntity<byte[]> embedReport(
            @RequestParam String path,
            @RequestParam Map<String, String> allParams) {

        allParams.put("format", "HTML5");
        // Reuse render logic
        return renderReport(path, "HTML5", allParams);
    }

    /**
     * List all available reports in a folder.
     */
    @GetMapping("/list")
    public ResponseEntity<String> listReports(
            @RequestParam(defaultValue = "/") String folder) {

        String url = REPORT_SERVER_URL.replace("/reportserver", "")
            + "/reports/api/v2.0/CatalogItems"
            + "?$filter=Path eq '" + folder + "'";

        RestTemplate restTemplate = createNtlmRestTemplate();
        ResponseEntity<String> response = restTemplate.getForEntity(url, String.class);
        return ResponseEntity.ok(response.getBody());
    }

    /**
     * Get report parameter definitions (so your UI can show the right inputs).
     */
    @GetMapping("/parameters")
    public ResponseEntity<String> getReportParameters(@RequestParam String path) {
        String url = REPORT_SERVER_URL.replace("/reportserver", "")
            + "/reports/api/v2.0/Reports(Path='" + path + "')/ParameterDefinitions";

        RestTemplate restTemplate = createNtlmRestTemplate();
        ResponseEntity<String> response = restTemplate.getForEntity(url, String.class);
        return ResponseEntity.ok(response.getBody());
    }

    /**
     * Create a RestTemplate configured for NTLM authentication.
     *
     * For production, use Apache HttpClient with NTLMScheme:
     *
     *   Add dependency: org.apache.httpcomponents.client5:httpclient5
     *
     *   Then configure CredentialsProvider with NTCredentials
     *   and use HttpComponentsClientHttpRequestFactory.
     */
    private RestTemplate createNtlmRestTemplate() {
        // SIMPLIFIED — for production, configure NTLM properly:
        //
        // 1. Add to pom.xml:
        //    <dependency>
        //      <groupId>org.apache.httpcomponents.client5</groupId>
        //      <artifactId>httpclient5</artifactId>
        //    </dependency>
        //
        // 2. Configure:
        //    BasicCredentialsProvider credsProvider = new BasicCredentialsProvider();
        //    credsProvider.setCredentials(
        //      new AuthScope(reportServerHost, 443),
        //      new NTCredentials(username, password, workstation, domain)
        //    );
        //    CloseableHttpClient httpClient = HttpClients.custom()
        //      .setDefaultCredentialsProvider(credsProvider)
        //      .build();
        //    HttpComponentsClientHttpRequestFactory factory =
        //      new HttpComponentsClientHttpRequestFactory(httpClient);
        //    return new RestTemplate(factory);

        return new RestTemplate();
    }
}
```


### React Frontend Implementation

Your React app calls your backend (not the Report Server directly):

```jsx
function ReportViewer() {
  const [year, setYear] = useState('2024');
  const [region, setRegion] = useState('');
  const [format, setFormat] = useState('HTML5');
  const [reportHtml, setReportHtml] = useState('');
  const [loading, setLoading] = useState(false);

  // Render as embedded HTML
  async function viewReport() {
    setLoading(true);
    const params = new URLSearchParams({
      path: '/Migrated Reports/Sales/Quarterly_Summary',
      format: 'HTML5',
      FISCAL_YEAR: year,
      REGION: region
    });

    const response = await fetch(`/api/reports/embed?${params}`);
    const html = await response.text();
    setReportHtml(html);
    setLoading(false);
  }

  // Download as PDF
  async function downloadPdf() {
    const params = new URLSearchParams({
      path: '/Migrated Reports/Sales/Quarterly_Summary',
      format: 'PDF',
      FISCAL_YEAR: year,
      REGION: region
    });

    const response = await fetch(`/api/reports/render?${params}`);
    const blob = await response.blob();
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'Quarterly_Summary.pdf';
    a.click();
  }

  // Download as Excel
  async function downloadExcel() {
    const params = new URLSearchParams({
      path: '/Migrated Reports/Sales/Quarterly_Summary',
      format: 'EXCELOPENXML',
      FISCAL_YEAR: year,
      REGION: region
    });

    const response = await fetch(`/api/reports/render?${params}`);
    const blob = await response.blob();
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'Quarterly_Summary.xlsx';
    a.click();
  }

  return (
    <div>
      <h2>Report Viewer</h2>

      {/* Parameter inputs */}
      <div>
        <label>Fiscal Year:
          <select value={year} onChange={e => setYear(e.target.value)}>
            <option value="2024">2024</option>
            <option value="2023">2023</option>
            <option value="2022">2022</option>
          </select>
        </label>

        <label>Region:
          <select value={region} onChange={e => setRegion(e.target.value)}>
            <option value="">All</option>
            <option value="North">North</option>
            <option value="South">South</option>
            <option value="East">East</option>
            <option value="West">West</option>
          </select>
        </label>
      </div>

      {/* Action buttons */}
      <div>
        <button onClick={viewReport}>View Report</button>
        <button onClick={downloadPdf}>Download PDF</button>
        <button onClick={downloadExcel}>Download Excel</button>
      </div>

      {/* Report display area */}
      {loading && <div>Loading report...</div>}
      {reportHtml && (
        <div
          dangerouslySetInnerHTML={{ __html: reportHtml }}
          style={{ border: '1px solid #ccc', padding: '20px', marginTop: '20px' }}
        />
      )}
    </div>
  );
}
```


## Method 3: Using the Report Server REST API v2.0

For more control, use the REST API directly.


### List All Reports

```
GET https://your-server/reports/api/v2.0/Reports
```

Response:
```json
{
  "value": [
    {
      "Id": "abc-123",
      "Name": "Quarterly_Summary",
      "Path": "/Migrated Reports/Sales/Quarterly_Summary",
      "Type": "Report",
      "HasParameters": true
    }
  ]
}
```


### Get Report Parameters

```
GET https://your-server/reports/api/v2.0/Reports(Path='/Migrated Reports/Sales/Quarterly_Summary')/ParameterDefinitions
```

Response:
```json
[
  {
    "Name": "FISCAL_YEAR",
    "ParameterType": "String",
    "DefaultValues": ["2024"],
    "Prompt": "Fiscal Year",
    "PromptUser": true,
    "ValidValues": [
      { "Label": "2024", "Value": "2024" },
      { "Label": "2023", "Value": "2023" }
    ]
  },
  {
    "Name": "REGION",
    "ParameterType": "String",
    "DefaultValues": ["ALL"],
    "Prompt": "Region",
    "PromptUser": true
  }
]
```

This is useful for dynamically building your UI's filter controls
based on what parameters each report actually has.


### Trigger Report Execution and Get Results

Render the report:
```
GET https://your-server/reportserver?/Migrated Reports/Sales/Quarterly_Summary
    &FISCAL_YEAR=2024&REGION=West&rs:Format=PDF&rs:Command=Render
```

This returns the rendered PDF as a binary stream.


## Method 4: SOAP API (for Advanced Scenarios)

Report Server also has a SOAP endpoint for programmatic control:

```
https://your-server/reportserver/ReportExecution2005.asmx
```

This allows:
- LoadReport (load a report definition)
- SetExecutionParameters (set parameter values)
- Render (execute and get output in specified format)
- ListChildren (list items in a folder)

This is more complex but gives you the most control. Microsoft
provides WSDL files you can use to generate client stubs in Java:

```
https://your-server/reportserver/ReportExecution2005.asmx?wsdl
```


## Authentication Options

### Option A: Windows Authentication (NTLM/Kerberos)

Default for on-prem. Your service account must be in Active Directory.

Java (Apache HttpClient 5):
```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
    <version>5.3</version>
</dependency>
```

```java
NTCredentials credentials = new NTCredentials(
    "service_account",     // username
    "password".toCharArray(),
    "WORKSTATION",         // your app server hostname
    "YOURDOMAIN"           // AD domain
);
```

### Option B: Custom Security Extension

If your org has configured a custom authentication extension
(forms-based auth), you'll authenticate via a login endpoint first,
get a cookie, and pass it with subsequent requests.

### Option C: Basic Auth (if configured)

Some orgs configure Report Server to accept Basic auth over HTTPS.
Simpler but less secure than NTLM:

```java
String auth = Base64.getEncoder().encodeToString(
    "DOMAIN\\username:password".getBytes());
HttpHeaders headers = new HttpHeaders();
headers.set("Authorization", "Basic " + auth);
```


## Putting It All Together: Complete Flow

### Your Backend Exposes These Endpoints

```
GET  /api/reports/list?folder=/Sales
     → Returns available reports with metadata

GET  /api/reports/parameters?path=/Sales/Quarterly_Summary
     → Returns parameter definitions for dynamic UI

GET  /api/reports/render?path=/Sales/Quarterly_Summary
         &format=PDF&FISCAL_YEAR=2024&REGION=West
     → Returns rendered PDF binary

GET  /api/reports/embed?path=/Sales/Quarterly_Summary
         &FISCAL_YEAR=2024&REGION=West
     → Returns rendered HTML for embedding
```

### Your Frontend Flow

```
1. Page loads
   → Call GET /api/reports/list to show available reports

2. User picks a report
   → Call GET /api/reports/parameters to get filter controls
   → Dynamically build dropdowns/inputs based on parameter definitions

3. User selects filters and clicks "View"
   → Call GET /api/reports/embed with parameters
   → Display rendered HTML inline

4. User clicks "Download PDF"
   → Call GET /api/reports/render with format=PDF
   → Browser downloads the file

5. User clicks "Download Excel"
   → Call GET /api/reports/render with format=EXCELOPENXML
   → Browser downloads the file
```


## Supported Export Formats

| Format Value | Output | Extension |
|---|---|---|
| `HTML5` | Interactive HTML | (inline) |
| `HTML4.0` | Legacy HTML | (inline) |
| `MHTML` | Web archive | .mhtml |
| `PDF` | PDF document | .pdf |
| `EXCELOPENXML` | Excel 2007+ | .xlsx |
| `EXCEL` | Excel 2003 | .xls |
| `CSV` | Comma-separated | .csv |
| `XML` | XML data | .xml |
| `IMAGE` | TIFF/PNG image | .tiff |
| `WORDOPENXML` | Word 2007+ | .docx |
| `PPTX` | PowerPoint | .pptx |


## Troubleshooting

**401 Unauthorized**
→ Your service account credentials are wrong or the account doesn't
  have permission on the Report Server. Test by opening the Report
  Server web portal in a browser with those credentials.

**Report renders but shows "no data"**
→ The parameter values you're passing don't match any data.
  Try the same parameters in the web portal first.

**NTLM authentication not working from Java**
→ Make sure you're using Apache HttpClient 5 with NTCredentials.
  The default Java HttpURLConnection doesn't support NTLM well.
  Add the `httpclient5` dependency and configure as shown above.

**Timeout on large reports**
→ Increase timeout on your RestTemplate and on Report Server's
  execution timeout (Report Server Configuration Manager →
  Execution → set longer timeout).

**HTML rendering looks broken when embedded**
→ The HTML returned by Report Server includes inline styles that
  may conflict with your app's CSS. Wrap it in an iframe or a
  shadow DOM to isolate styles:
  ```html
  <iframe srcdoc={reportHtml} width="100%" height="800px" />
  ```

**Cross-origin issues**
→ Your app backend should proxy all Report Server calls. Never
  call Report Server directly from the browser unless both are
  on the same domain.
