# Dalgo Platform

## 🔄 Development Workflow

### 📋 Feature Development Process

1. **📝 Write Specification**
   - Create a detailed spec for the feature or enhancement
   - Save to: `dalgo_mds/specs/<feature_name>.md`

2. **🛠️ Generate Implementation Plan**
   - Use Claude command to generate detailed implementation plan
   - Command: `/plan-feature dalgo_mds/specs/<feature_name>.md`

3. **⚡ Execute Plan**
   - Execute the generated plan using Claude command
   - Command: `/execute-plan dalgo_mds/planning/<feature_name>_plan.md`

