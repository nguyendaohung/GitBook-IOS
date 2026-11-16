# 🛠 Testing, Performance và Release

* [Unit Test và UI Test](testing.md)
* [Instruments và Production Debugging](performance.md)
* [Dependency, CI/CD và App Store](release.md)
* [Localization, Security và AI-assisted Development](production-practices.md)

## Câu hỏi phỏng vấn tổng quan

**Q: Unit/integration/UI test chọn thế nào?** Unit logic isolated, integration boundary DB/API, UI user flow; fake dependencies/clock, device test extension/watch/VPN.

**Q: Tối ưu performance từ đâu?** Reproduce, measure baseline, profile bottleneck, fix và measure lại; không đoán.

**Q: Release iOS cần gì?** Signing/capability, dependencies, CI tests/archive/upload, TestFlight, privacy/review, monitoring và hotfix plan.
