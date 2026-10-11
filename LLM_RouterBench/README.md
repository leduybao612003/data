Bench LLMRouterBench cho eval PRD-5.1

## Cấu trúc

| Đường dẫn | Nội dung |
|---|---|
| `_src/` | Sparse checkout repo `ynulihao/LLMRouterBench` @ `c77cb05` (đủ `data/`, `data_collector/`, `evaluation/`, `config/`) |
| `prd51_eval/` | 7 slice × 200 mẫu = 1400 dòng JSONL + `manifest.json` (seed 42) |
| `prd51_eval/READ_ME_KIEM_TRA.txt` | Mapping slice↔dataset + giới hạn đã biết (proxy, dedup GPQA) |

## Yêu cầu

- Python 3.10+ (máy hiện tại 3.13 OK), MySQL **optional** (chỉ cho cache).
- API key provider (khuyến nghị `OPENROUTER_API_KEY` vì 1 key gọi nhiều model). → Chuyển sang dùng Key từ gateway

## 1. Cài đặt (PowerShell)

```powershell
cd E:\data\_src
python -m venv .venv; .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env   # mở .env điền key
```

Không có MySQL: đặt `cache.enabled: false` trong file config yaml.

## 2. Config nào dùng khi nào

| File | Dùng khi nào |
|---|---|
| `config/data_collector_small_model_config.yaml` | Bench performance (model ~7B, rẻ) |
| `config/data_collector_proprietary_model_config.yaml` | Bench performance-cost (flagship, tốn $) |

3 khối trong mỗi config: `models` (name/base_url/api_key/pricing) → `datasets`
(dataset_id + splits) → `run` (`concurrency`, `demo_mode`, `output_dir`).

## 3. Chạy thử 10 mẫu (bắt buộc trước khi chạy thật)

```powershell
python -m data_collector.cli info config/data_collector_small_model_config.yaml
# bật run.demo_mode=true, demo_limit=10 trong yaml rồi:
python -m data_collector.cli run config/data_collector_small_model_config.yaml
python -m data_collector.cli list
```

Kết quả: `results/bench/<dataset>/<split>/<model>/<timestamp>.json`
(`performance, time_taken, records[]{origin_query, prompt, prediction,
ground_truth, score, *_tokens, cost}`).

## 4. Chạy thật

```powershell
# tắt demo_mode, để concurrency=8 trước (tránh 429), rồi mới tăng
python -m data_collector.cli run config/data_collector_small_model_config.yaml
# chạy lại từ đầu (bỏ cache):
python -m data_collector.cli run config/data_collector_small_model_config.yaml --force
```

Mất mạng giữa chừng: chạy lại lệnh run, cache MySQL (nếu bật) bỏ qua mẫu đã xong.

## 5. Phân tích routing (Gain/Gap/Pareto)

```python
from baselines import BaselineDataLoader, BaselineAggregator
loader = BaselineDataLoader("config/baseline_config.yaml")
records = loader.load_all_records()
agg = BaselineAggregator(records, data_loader=loader)
agg.print_summary_tables(score_as_percent=True, test_mode=False)
```

## 6. Eval gateway (khuyến nghị, rẻ hơn chạy full bench)

Dùng sẵn `prd51_eval/*.jsonl`, replay `origin_query` qua
`POST /routing-policies/{id}/simulate`, so với `ground_truth` + oracle:

- Slice `03_hard_policy`: gắn thêm `policy_overlay` mỗi dòng, đo hard-policy
  violation rate (phải 0% mới release).
- Slice `05_provider_failure`: tiêm `fault_plan` (timeout/429/5xx/unavailable)
  rồi đo fallback success rate (≥ 95%).

## 7. Đo latency concurrent (cách B đã chốt)

- Baseline `concurrency=1`, rồi `8/32`; mỗi mức lặp 3 lần.
- Đo provider thật: `force_override_cache=true`; đo prod có cache: `false`.
- SLO = `p95_provider + classifierMs (≤1500 fail-soft) + gatewayOverheadMs (p99 15ms) + guardrailMs`.


