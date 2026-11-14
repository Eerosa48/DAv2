# DAv2 Development Rules

1. DAv2 must be completely independent of Disney App 1.0. Never modify, delete, or write files in the V1 repository or its local folder.
2. All code, assets, map tiles, dependencies, and required data must be contained within DAv2 or use independently configured external APIs. Never fetch resources from the V1 website or repository.
3. The architecture should be modular, with reusable components, clearly defined responsibilities, and minimal coupling between modules.
4. UI controls, panels, map overlays, and notifications must use deliberate, centrally managed layering and interaction rules.
5. Preserve existing functional behavior while refactoring, unless a change is explicitly approved.
6. Maintain responsive compatibility with desktop, iPhone Safari, installed iPhone PWA, tablets, and foldable displays.
7. V2 must use independent service-worker caches, browser storage identifiers, and properly scoped paths that cannot conflict with V1.
8. Keep resource paths portable so the finished V2 application can later be deployed under the original V1 GitHub Pages path without depending on the DAv2 URL.
9. Prioritize performance, stability, maintainability, accessibility, and offline functionality.
10. Before significant code changes, explain the intended changes and their impact. Run relevant checks or tests after implementation, and report their results.
11. Do not commit, push, deploy, delete significant files, or modify Git configuration without explicit approval.
12. Work incrementally in small, verifiable steps. Do not perform unrelated refactoring or introduce unnecessary dependencies.
