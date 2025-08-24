 extraction_result.jsonは以下の3つの主要なタイミングで生成されます：

  1️⃣ UnifiedQualityChecker による品質評価時

  # tools/core/unified_quality_checker.py
  class UnifiedQualityChecker:
      def run_quality_check(self, extraction_dir: Path) -> Dict[str, Any]:
          """品質チェック実行とextraction_result.json生成"""
          # 抽出結果の品質評価
          results = self.check_extraction_results(extraction_dir)

          # extraction_result.jsonとして保存
          output_path = extraction_dir / "extraction_result.json"
          with open(output_path, 'w', encoding='utf-8') as f:
              json.dump(results, f, ensure_ascii=False, indent=2)

  2️⃣ 統計分析システムによる自動生成

  # deprecated/progress_tracker_legacy/universal_statistical_analyzer.py
  def ensure_extraction_result_exists(extraction_dir: Path) -> Dict:
      """extraction_result.jsonが存在しない場合は生成"""
      result_path = extraction_dir / "extraction_result.json"

      if not result_path.exists():
          # UnifiedQualityCheckerを使用して生成
          quality_checker = UnifiedQualityChecker()
          results = quality_checker.run_quality_check(extraction_dir)

          with open(result_path, 'w') as f:
              json.dump(results, f, ensure_ascii=False, indent=2)

  3️⃣ 品質ワークフロー実行時

  # tools/scripts/run_quality_workflow.sh
  # Step 3: 品質評価実行
  python tools/core/unified_quality_checker.py \
      --extraction-dir "${EXTRACTION_DIR}" \
      --output "${EXTRACTION_DIR}/extraction_result.json"

  📋 ファイル内容の構造

  extraction_result.jsonには以下の情報が含まれます：

  {
    "summary": {
      "total_images": 20,
      "successful_extractions": 18,
      "success_rate": 0.9,
      "average_quality_score": 0.742
    },
    "quality_distribution": {
      "A": 5,
      "B": 8,
      "C": 3,
      "D": 2,
      "E": 0,
      "F": 0
    },
    "individual_results": [
      {
        "filename": "image_001.jpg",
        "status": "success",
        "quality_score": 0.856,
        "quality_grade": "A",
        "metrics": {
          "mask_coverage": 0.324,
          "edge_clarity": 0.892,
          "boundary_smoothness": 0.765
        }
      }
    ],
    "statistical_metrics": {
      "mean": 0.742,
      "std": 0.123,
      "median": 0.756,
      "min": 0.482,
      "max": 0.938
    }
  }

  🔄 生成トリガーの流れ

  graph TD
      A[画像抽出処理完了] --> B{extraction_result.json存在?}
      B -->|No| C[UnifiedQualityChecker実行]
      B -->|Yes| D[既存ファイル使用]
      C --> E[品質評価実施]
      E --> F[extraction_result.json生成]
      F --> G[ダッシュボード生成]
      D --> G
      G --> H[統計分析レポート作成]

  💡 重要な発見

  1. 自動生成メカニズム: 統計分析システムがextraction_result.jsonを必要とする際、存在しない場合は自動的にUnifiedQualityCheckerを呼び出して生成
  2. 品質保証の中核: このファイルが品質評価システムの中心的なデータソースとして機能
  3. 再生成可能: ファイルが削除されても、抽出画像が存在すれば自動的に再生成される設計

  この仕組みにより、品質評価データの一貫性と可用性が保証されています。



 1️⃣ UnifiedQualityChecker による品質評価時
   # tools/core/unified_quality_checker.py

 2️⃣ 統計分析システムによる自動生成
   # deprecated/progress_tracker_legacy/universal_statistical_analyzer.py
   
3️⃣ 品質ワークフロー実行時
  # tools/scripts/run_quality_workflow.sh

この3つのPythonの実行はそれぞれ同じ extract_result.json を出力するのでしょうか。
それとも一つ一つがそれぞれ必要なextract_result.jsonを出力してしまってコマンドのPythonによって内容の差異が生じてしまうものでしょうか

  | 実行パス                       | 品質スコア算出方法 | 画像分析             | 統計機能    | ファイル形式   |  
  |-----------------------|---------------------|-----------------|------------|---------------|
  | UnifiedQualityChecker | OpenCV実測値        | ✅ リアルタイム | ❌                | 詳細JSON        |
  | Statistical Analyzer       | UQCに委譲              | ✅ UQC経由       | ✅ 比較分析 | 統計特化JSON |    
  │ | Quality Workflow         | 固定推定値              | ❌ 推定のみ        | ❌                  | 簡易JSON   |      



│
 deprecated/progress_tracker_legacy/universal_statistical_analyzer.py
これらのプログラムは非推奨のためそのうち削除される予定なのでこれはまあいいとして

Quality Workflowの存在は問題があります
unified_quality_checker.py　このプログラムが実行される時は どのようなパターンやワーグフローが実行されますでしょうか?ドキュメントや実装をみて調査してみて


