# LungPath AI — Roboflow YOLO Object-Detection Starter

This project connects a static HTML/CSS/JavaScript frontend to a FastAPI backend and a YOLO object-detection model. It is prepared for the public Roboflow Universe project **Ahills Workspace / cancer-cell-0smsb, version 1**.

## Important dataset/task check

The linked Roboflow project is listed as **Object Detection**, not binary image classification. Its version 1 page reports 551 examples (448 train, 64 validation, 39 test) and provides YOLO export formats. The overview page separately displays 327 images and unclear class metadata. The downloaded `data.yaml` and actual sample labels must therefore be inspected before training.

This means the model produced by the included notebook will detect the objects annotated in that dataset and return bounding boxes. It will **not** automatically classify a whole lung histopathology image as `Tumor` or `Normal`, and the project page alone does not establish that the images are lung tissue. If the project's required output remains Tumor-vs-Normal lung-tissue classification, use a dataset with explicit patient-level `Normal` and `Tumor` image labels instead of treating this object-detection dataset as interchangeable.

Dataset license shown on Roboflow Universe: CC BY 4.0. Keep attribution to Ahills Workspace and comply with the license when redistributing or publishing derived work.

## 1. Train the model in Google Colab

1. Open `training/Train_LungPath_Cancer_Cell_Detector.ipynb` in Google Colab.
2. In Colab **Secrets**, add `ROBOFLOW_API_KEY` and enable notebook access. Do not put the secret in source code or GitHub.
3. Run the notebook from top to bottom. It downloads Roboflow version 1 in YOLOv8 format, prints the data configuration and class names, displays sample annotations, trains YOLOv8n, and evaluates on the test split.
4. Download the resulting `best.pt` and place it at `model/best.pt` in this project.
5. Do not present training as complete until the notebook has actually run and the test metrics have been reviewed.

The training notebook retains Roboflow's published train/validation/test split. If patient identifiers exist, check whether images from the same patient cross splits; patient-level separation is needed for a credible independent evaluation.

## 2. Run locally

Use the backend environment you already created, then from the `backend` directory run:

```powershell
python -m pip install -r requirements.txt
python -m uvicorn main:app --reload
```

Open `http://127.0.0.1:8000` and API docs at `http://127.0.0.1:8000/docs`. The website and API can still run on one local URL. Until `model/best.pt` is installed, `/api/health` reports `model_ready: false` and `/api/analyze` returns HTTP 503 after saving the upload; it will not invent a prediction.

## 3. Recommended deployment: Vercel frontend + Railway backend/model

Keep the frontend and inference API on separate services:

- **Vercel:** deploy the `frontend/` directory as a static site.
- **Railway:** deploy the project root using the included `Dockerfile`. Railway runs FastAPI, loads `model/best.pt`, and exposes `/api/health`, `/api/analyze`, and `/api/analyses`.

This is the recommended layout for this PyTorch/YOLO app. Vercel can host FastAPI, but its functions have bundle, request-body, and execution-time limits; a long-lived Railway service is a more natural place for an AI model that is loaded once and reused across requests.

### Railway setup

1. Push this project to a private GitHub repository.
2. In Railway, create a project and deploy from that repository (Dockerfile is in the repository root).
3. Add a persistent Railway Volume mounted at `/app/data`. The SQLite database and uploaded images are saved there by default. For a multi-replica/production system, use PostgreSQL plus object storage rather than a shared SQLite file.
4. Add environment variables:

   - `DATA_DIR=/app/data`
   - `CORS_ORIGINS=https://YOUR-VERCEL-DOMAIN.vercel.app`
   - `MODEL_PATH=/app/model/best.pt` (default path; set differently only if storing weights on a volume)
   - `CONFIDENCE_THRESHOLD=0.25`
   - `IMAGE_SIZE=640`
   - `MODEL_DEVICE=cpu`

5. Make sure the trained `best.pt` is present in `model/` before building the Railway image. If the weights are too large for GitHub, upload them to a persistent Railway volume and set `MODEL_PATH` to the mounted path.
6. After deployment, open `https://YOUR-RAILWAY-DOMAIN/api/health`. It should return `model_ready: true` only when the weight file exists.

### Vercel setup

1. Import the same GitHub repository into Vercel.
2. Set **Root Directory** to `frontend` and deploy it as a static site (no build command is needed).
3. Edit `frontend/config.js` before deployment and set:

   ```js
   window.LUNGPATH_API_URL = "https://YOUR-RAILWAY-DOMAIN";
   ```

4. Copy the exact production Vercel domain into Railway's `CORS_ORIGINS` variable and redeploy the Railway service.
5. Test the Vercel site: backend status should show connected and model status should show ready. Upload a supported sample image and check the response and saved history.

### Database and storage note

The included app supports SQLite (default) or PostgreSQL through `DATABASE_URL`. If using PostgreSQL, add a Railway PostgreSQL service and set the provided connection URL as `DATABASE_URL`. A Railway volume is still useful for uploaded image files. Never assume local container storage persists across deployments.

## API endpoints

- `GET /api/health` — backend and model file status.
- `POST /api/analyze` — multipart upload with form field `file`; returns a summary and normalized bounding-box detections.
- `GET /api/analyses` — latest saved analysis records.
- `GET /docs` — interactive API documentation.

## Research and safety statement

LungPath AI is an educational research prototype. Object detections and confidence scores are not clinical diagnoses, and must not be used to make patient-care decisions. Before any clinical use, the dataset provenance, tissue type, labeling quality, patient-level split, external validation, bias, calibration, and clinical requirements would need independent review.
