<div align="center">
  <h1>Adobe</h1>
  <small>Adobe Document Services API client for Rust</small>
</div>

## Installation

### Cargo

```bash
cargo add adobe
```

### GitHub Releases

You can also download precompiled binaries from the [GitHub Releases](https://github.com/LeoBorai/adobe/releases) page.

## How to Use

### Prerequisites

You will need an **Adobe PDF Services** account with a **Client ID** and **Client Secret**.
You can obtain these credentials from the [Adobe Developer Console](https://developer.adobe.com/console).

### Setting Up the Client

Create an instance of `PdfServicesClient` by providing the API base URL, your Client ID, and Client Secret.
The client handles authentication automatically.

```rust
use adobe::pdf::{PDF_SERVICES_UE1_URL, PdfServicesClient};

#[tokio::main]
async fn main() {
    let client = PdfServicesClient::new(
        PDF_SERVICES_UE1_URL,
        "YOUR_CLIENT_ID",
        "YOUR_CLIENT_SECRET",
    )
    .await
    .expect("Failed to create PDF Services client");
}
```

Two base URL constants are provided:

- `PDF_SERVICES_UE1_URL` — US East region (`https://pdf-services-ue1.adobe.io`)
- `PDF_SERVICES_EW1_URL` — Europe West region (`https://pdf-services-ew1.adobe.io`)

### Uploading a PDF

Before running any operation, upload your PDF to obtain an asset ID:

```rust
use bytes::Bytes;
use adobe::pdf::{PDF_SERVICES_UE1_URL, PdfServicesClient};
use adobe::pdf::request::assets::get_upload_presigned_uri::GetUploadPresignedUriParams;

#[tokio::main]
async fn main() {
    let client = PdfServicesClient::new(
        PDF_SERVICES_UE1_URL,
        "YOUR_CLIENT_ID",
        "YOUR_CLIENT_SECRET",
    )
    .await
    .unwrap();

    let pdf_bytes = Bytes::from(tokio::fs::read("path/to/file.pdf").await.unwrap());

    // Request a pre-signed URI for uploading
    let upload_info = client
        .get_upload_pre_signed_uri(GetUploadPresignedUriParams {
            media_type: String::from("application/pdf"),
        })
        .await
        .unwrap();

    // Upload the PDF file
    client
        .upload_pdf_file(upload_info.upload_uri, pdf_bytes)
        .await
        .unwrap();

    // Use upload_info.asset_id in subsequent operations
    println!("Asset ID: {}", upload_info.asset_id);
}
```

### Performing OCR

Convert a scanned PDF into a searchable PDF using the OCR operation:

```rust
use bytes::Bytes;
use adobe::pdf::{PDF_SERVICES_UE1_URL, PdfServicesClient};
use adobe::pdf::request::assets::get_upload_presigned_uri::GetUploadPresignedUriParams;
use adobe::pdf::request::operations::ocr::{OcrLang, OcrParams, OcrType};
use adobe::pdf::request::operations::ocr_poll::{OcrJobStatus, OcrPollParams};

#[tokio::main]
async fn main() {
    let client = PdfServicesClient::new(
        PDF_SERVICES_UE1_URL,
        "YOUR_CLIENT_ID",
        "YOUR_CLIENT_SECRET",
    )
    .await
    .unwrap();

    let pdf_bytes = Bytes::from(tokio::fs::read("path/to/scanned.pdf").await.unwrap());

    // Upload the PDF
    let upload_info = client
        .get_upload_pre_signed_uri(GetUploadPresignedUriParams {
            media_type: String::from("application/pdf"),
        })
        .await
        .unwrap();

    client
        .upload_pdf_file(upload_info.upload_uri, pdf_bytes)
        .await
        .unwrap();

    // Start the OCR job
    let ocr = client
        .perform_ocr(OcrParams {
            asset_id: upload_info.asset_id,
            ocr_lang: OcrLang::EnUS,
            ocr_type: OcrType::SearchableImage,
        })
        .await
        .unwrap();

    // Poll until the job is complete
    loop {
        let poll = client
            .poll_ocr(OcrPollParams {
                job_id: ocr.job_id.clone(),
            })
            .await
            .unwrap();

        match poll.status {
            OcrJobStatus::Done => {
                let asset = poll.asset.unwrap();
                println!("OCR complete! Download URI: {:?}", asset.download_uri);
                break;
            }
            OcrJobStatus::Failed => panic!("OCR job failed"),
            OcrJobStatus::InProgress => {
                println!("OCR in progress, waiting...");
                tokio::time::sleep(std::time::Duration::from_secs(2)).await;
            }
        }
    }
}
```

### Extracting PDF Content

Extract structured text, tables, and other elements from a PDF document:

```rust
use bytes::Bytes;
use adobe::pdf::{PDF_SERVICES_UE1_URL, PdfServicesClient};
use adobe::pdf::request::assets::get_upload_presigned_uri::GetUploadPresignedUriParams;
use adobe::pdf::request::operations::extract::ExtractParams;
use adobe::pdf::request::operations::extract_poll::{ExtractJobStatus, ExtractPollParams};

#[tokio::main]
async fn main() {
    let client = PdfServicesClient::new(
        PDF_SERVICES_UE1_URL,
        "YOUR_CLIENT_ID",
        "YOUR_CLIENT_SECRET",
    )
    .await
    .unwrap();

    let pdf_bytes = Bytes::from(tokio::fs::read("path/to/document.pdf").await.unwrap());

    // Upload the PDF
    let upload_info = client
        .get_upload_pre_signed_uri(GetUploadPresignedUriParams {
            media_type: String::from("application/pdf"),
        })
        .await
        .unwrap();

    client
        .upload_pdf_file(upload_info.upload_uri, pdf_bytes)
        .await
        .unwrap();

    // Start the extract job
    let extract = client
        .extract_pdf(ExtractParams {
            asset_id: upload_info.asset_id,
            get_char_bounds: false,
            include_styling: false,
            elements_to_extract: vec!["text".to_string(), "tables".to_string()],
        })
        .await
        .unwrap();

    // Poll until the job is complete
    loop {
        let poll = client
            .poll_extract_pdf(ExtractPollParams {
                job_id: extract.job_id.clone(),
            })
            .await
            .unwrap();

        match poll.status {
            ExtractJobStatus::Done => {
                let content = poll.content.unwrap();
                println!("Extraction complete! Download URI: {:?}", content.download_uri);
                break;
            }
            ExtractJobStatus::Failed => panic!("Extract job failed"),
            ExtractJobStatus::InProgress => {
                println!("Extraction in progress, waiting...");
                tokio::time::sleep(std::time::Duration::from_secs(2)).await;
            }
        }
    }
}
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE.md) file for details.
This project is also licensed under the Apache License 2.0 - see the [LICENSE-APACHE](LICENSE-APACHE.md) file for details.
