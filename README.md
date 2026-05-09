import cv2
import numpy as np

# =========================================================
# 1. RECIPE (CONFIGURATION)
# =========================================================
RECIPE = {
    "models": {
        "anomaly_threshold": 0.6,
        "confidence_threshold": 0.85
    }
}

# =========================================================
# 2. CAMERA MODULE
# =========================================================
class Camera:
    def __init__(self, src=0):
        self.cap = cv2.VideoCapture(src)

    def get_frame(self):
        ret, frame = self.cap.read()
        if not ret:
            return None
        return frame

    def release(self):
        self.cap.release()

# =========================================================
# 3. VISION PROCESSING (SIMULATED YOLO + DEFECT DETECTION)
# =========================================================
class VisionSystem:
    def detect_defects(self, frame):
        """
        Simulated defect detection using contour + noise logic
        (Replace with YOLOv8 in real system)
        """

        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        blurred = cv2.GaussianBlur(gray, (5, 5), 0)

        edges = cv2.Canny(blurred, 50, 150)

        contours, _ = cv2.findContours(edges, cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)

        detections = []

        for c in contours:
            area = cv2.contourArea(c)

            if area > 500:  # defect-like region threshold
                x, y, w, h = cv2.boundingRect(c)

                confidence = min(1.0, area / 5000)

                detections.append({
                    "bbox": [x, y, x + w, y + h],
                    "confidence": confidence
                })

                cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 0, 255), 2)

        return frame, detections

# =========================================================
# 4. ANOMALY DETECTION (SIMPLIFIED PATCHCORE-LIKE)
# =========================================================
def anomaly_score(frame):
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    score = np.std(gray) / 255.0
    return float(score)

# =========================================================
# 5. DECISION ENGINE
# =========================================================
def decision_engine(detections, anomaly):
    threshold = RECIPE["models"]["anomaly_threshold"]
    conf_thres = RECIPE["models"]["confidence_threshold"]

    reasons = []

    if anomaly > threshold:
        reasons.append("HIGH_ANOMALY")

    if len(detections) == 0:
        reasons.append("MISSING_COMPONENT")

    for d in detections:
        if d["confidence"] < conf_thres:
            reasons.append("LOW_CONFIDENCE_DEFECT")

    if len(reasons) == 0:
        return "PASS", reasons
    else:
        return "FAIL", reasons

# =========================================================
# 6. PLC SIMULATION
# =========================================================
class PLC:
    def send(self, result):
        if result == "PASS":
            print("[PLC] PASS → DO-01 signal sent")
        else:
            print("[PLC] FAIL → DO-02 reject activated")

# =========================================================
# 7. MAIN PIPELINE
# =========================================================
def main():
    cam = Camera(0)
    vision = VisionSystem()
    plc = PLC()

    print("ECI-Vision System Started... Press 'q' to quit")

    while True:
        frame = cam.get_frame()

        if frame is None:
            break

        # Step 1: Detection
        processed_frame, detections = vision.detect_defects(frame)

        # Step 2: Anomaly score
        score = anomaly_score(frame)

        # Step 3: Decision
        result, reasons = decision_engine(detections, score)

        # Step 4: PLC output
        plc.send(result)

        # Step 5: UI Overlay
        cv2.putText(processed_frame,
                    f"RESULT: {result}",
                    (30, 40),
                    cv2.FONT_HERSHEY_SIMPLEX,
                    1,
                    (0, 255, 0) if result == "PASS" else (0, 0, 255),
                    2)

        cv2.putText(processed_frame,
                    f"Anomaly: {score:.2f}",
                    (30, 80),
                    cv2.FONT_HERSHEY_SIMPLEX,
                    0.8,
                    (255, 255, 0),
                    2)

        # Step 6: Show reasons
        y = 120
        for r in reasons:
            cv2.putText(processed_frame,
                        r,
                        (30, y),
                        cv2.FONT_HERSHEY_SIMPLEX,
                        0.6,
                        (0, 0, 255),
                        2)
            y += 25

        cv2.imshow("ECI-Vision Inspection System", processed_frame)

        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    cam.release()
    cv2.destroyAllWindows()

# =========================================================
# 8. RUN
# =========================================================
if __name__ == "__main__":
    main()
