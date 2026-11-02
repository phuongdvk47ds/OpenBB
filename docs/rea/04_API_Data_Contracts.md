# 04 — API Data Contracts

| Thuộc tính | Giá trị |
|---|---|
| Phạm vi | Mô hình Pydantic và dataclass dùng để validate request/response của hệ thống API (REST, SDK, MCP, Platform API) |
| Phương pháp | Quét AST toàn bộ `openbb_platform/**` (bỏ `tests`), đọc `provider_interface.py`, `router.py`, `exception_handlers.py`, `commands.py` |
| Tài liệu liên quan | `02_Architecture_C4.md`, `03_Core_Business_Logic.md`, `docs/WF_SRS.md` |
| Ký hiệu | **R** bắt buộc; **O** tùy chọn (có mặc định); `[OK]` xác minh trong mã; `[!]` bất thường; `[UNKNOWN]` không xác định |

---

## 1. Tổng quan

### 1.1 Thống kê định nghĩa mô hình (lớp cấp module, ngoài `tests`)

| Khu vực | `QueryParams` | `Data` | `BaseModel` khác | `dataclass` |
|---|---|---|---|---|
| `core` | 180 | 187 | 24 | 9 |
| `extensions` | 101 | 106 | 14 | 1 |
| `obbject_extensions` | 46 | — | — | 3 |
| `providers` | 780 | 797 | 3 | 9 |

Lõi có 181 tệp trong `core/openbb_core/provider/standard_models/` định nghĩa cặp `<Model>QueryParams` / `<Model>Data`. Provider kế thừa lớp chuẩn và bổ sung trường riêng.

### 1.2 Phân lớp hợp đồng

| Lớp | Vai trò | Mô hình chính |
|---|---|---|
| **L1 Request** | Tham số vào lệnh | `StandardParams`, `ExtraParams`, `ProviderChoices` (dataclass sinh động); `QueryParams` (lệnh `POST` định lượng); `AgGridRowsRequest` |
| **L2 Response** | Phong bì kết quả | `OBBject[T]`, `AnnotatedResult[T]`, `OBBStream`, `Data` |
| **L3 Error** | Phản hồi lỗi | `OpenBBErrorResponse` (khai báo OpenAPI), thân thực tế `{"detail": ...}` |
| **L4 Configuration** | Cấu hình đi kèm yêu cầu | `UserSettings`, `Credentials`, `Preferences`, `Defaults`, `SystemSettings`, `APISettings`, `CommandContext` |
| **L5 Integration** | Hợp đồng kênh | Spec Platform API/MCP (`_SpecDocumentModel`), `MCPConfigModel`, `MCPSettings`, widget response models |

### 1.3 Cơ chế sinh hợp đồng động `[OK]`

`ProviderInterface` (`app/provider_interface.py`, singleton) dựng hợp đồng lúc khởi động từ các provider đã cài:

| Bước | Hành động | Dòng |
|---|---|---|
| 1 | `make_dataclass` tạo `StandardParams` và `ExtraParams` cho từng model (trường chuẩn và trường riêng từng provider) | 538-543 |
| 2 | `make_dataclass` tạo `ProviderChoices` với `provider: Literal[<tên provider hỗ trợ model>]` | 570-575, 680-682 |
| 3 | `create_model` tạo `StandardData` / `ExtraData` | 627-632, 671 |
| 4 | Dựng kiểu trả về `OBBject[SerializeAsAny[Annotated[Union[<Data theo provider, Tag(provider)>], discriminator]]]` | 698-705 |

Hệ quả: hợp đồng request/response của mỗi lệnh phụ thuộc tập provider đã cài; cùng một mã nguồn có thể tạo schema OpenAPI khác nhau.

---

## 2. Data Contracts cốt lõi (L2 — Response)

### 2.1 `OBBject[T]` — phong bì kết quả (`app/model/obbject.py:34`)

Kế thừa `Tagged` (trường `id: str`, mặc định `uuid7str`).

| Trường | Kiểu | R/O | Mặc định | Mô tả / ràng buộc |
|---|---|---|---|---|
| `id` | `str` | O | `uuid7str()` | Mã định danh phiên bản (từ `Tagged`) |
| `results` | `T \| None` | O | `None` | Dữ liệu kết quả; thường `list[<Model>Data]` |
| `provider` | `str \| None` | O | `None` | Provider đã phục vụ |
| `warnings` | `list[Warning_] \| None` | O | `None` | Cảnh báo ghi nhận khi thực thi |
| `chart` | `Chart \| None` | O | `None` | Biểu đồ khi `chart=true` |
| `extra` | `dict[str, Any]` | O | `{}` | Chứa `metadata`, `results_metadata` |
| `_route`, `_standard_params`, `_extra_params` | PrivateAttr | — | `None` | Trạng thái nội bộ; không tuần tự hóa |

### 2.2 Mô hình con của phong bì

| Mô hình | Nguồn | Trường |
|---|---|---|
| `Warning_` | `model/abstract/warning.py` | `category: str` (R), `message: str` (R) |
| `Chart` | `model/charts/chart.py` | `content: dict \| None`, `format: str \| None`, `fig: Any \| None` (đều O) |
| `Metadata` | `model/metadata.py` | `arguments: dict` (O), `duration: int` (R, ns), `route: str` (R), `timestamp: datetime` (R) |
| `AnnotatedResult[T]` | `provider/abstract/annotated_result.py` | `result: T \| None`, `metadata: dict \| None` (đều O); `metadata` vào `OBBject.extra["results_metadata"]` |

### 2.3 `Data` — bản ghi chuẩn (`provider/abstract/data.py`)

| Thuộc tính | Giá trị |
|---|---|
| `extra` | `allow` — nhận trường ngoài khai báo |
| `populate_by_name` | `True` |
| `strict` | `False` |
| `validation_alias` | `to_camel` — nhận khóa camelCase |
| `serialization_alias` | `to_snake` — xuất snake_case |
| `__alias_dict__` | Ánh xạ tên gốc của provider sang tên chuẩn, đảo ngược trong `_use_alias` |
| Kiểu hỗ trợ | `ForceInt` (ép `int`, lỗi `TypeError`) |

### 2.4 `OBBStream` — phản hồi luồng (`app/model/stream.py:75`)

Lớp thường (không phải Pydantic). Thuộc tính: `id`, `provider`, `warnings`, `extra`, `media_type`. Phương thức: `start`, `stop`, `wait`, lặp bất đồng bộ. Trên REST, thân là `text/event-stream`; `id`, `provider`, `warnings` truyền qua header `X-OpenBB-Stream-Id`, `X-OpenBB-Provider`, `X-OpenBB-Warning` (`api/router/commands.py`) `[OK]`.

### 2.5 Ví dụ hợp đồng dữ liệu chuẩn: Equity Historical (`standard_models/equity_historical.py`)

| Mô hình | Trường | Kiểu | R/O | Ràng buộc |
|---|---|---|---|---|
| `EquityHistoricalQueryParams` | `symbol` | `str` | R | Validator `to_upper` (chữ hoa) |
| | `start_date`, `end_date` | `date \| None` | O | — |
| `EquityHistoricalData` | `date` | `date \| datetime` | R | Chuỗi có `:` → `datetime` ISO 8601, ngược lại `date` |
| | `open`, `high`, `low`, `close` | `float` | R | — |
| | `volume` | `float \| int \| None` | O | — |
| | `vwap` | `float \| None` | O | — |

---

## 3. Data Contracts Request (L1)

### 3.1 Request chuẩn cho lệnh dữ liệu (GET)

| Thành phần | Kiểu | Sinh bởi | Nội dung |
|---|---|---|---|
| `provider_choices` | dataclass `ProviderChoices` | `make_dataclass` | `provider: Literal[...]` |
| `standard_params` | dataclass `StandardParams` | `make_dataclass` | Trường của `<Model>QueryParams` chuẩn |
| `extra_params` | dataclass `ExtraParams` | `make_dataclass` | Trường riêng provider; mỗi trường có `title` là danh sách provider hỗ trợ (dùng cho `Query.filter_extra_params`) |
| `cc` | `CommandContext` | Core | Không đưa vào schema API (`build_new_signature` bỏ qua) |
| `chart` | `bool` | Core | Chỉ thêm khi charting extension có hàm cho route |
| Header tùy chỉnh | `str` | `api_settings.custom_headers` | Giá trị mặc định lấy từ cấu hình |

### 3.2 Request cho lệnh định lượng (POST)

Lệnh `@router.command(methods=["POST"])` nhận **một** tham số `params: <X>QueryParams` làm thân JSON. Quy tắc chung `[OK]`:

| Trường | Kiểu | R/O | Ghi chú |
|---|---|---|---|
| `data` | `list[Data]` | R | Bộ dữ liệu đầu vào dạng bản ghi |
| `target` | `str` | R (trừ hồi quy nhân tố: O) | Tên cột phân tích |
| `index` | `str` | O (`"date"`) | Cột chỉ mục |
| `window` | `PositiveInt` | O (`252` cho Sharpe/Sortino) | Cửa sổ cuốn chiếu |

Hợp đồng theo lệnh (`extensions/quantitative/openbb_quantitative`):

| Lệnh | QueryParams → Data | Trường bổ sung (QueryParams) | Trường Data |
|---|---|---|---|
| `sharpe_ratio` | `SharpeRatioQueryParams` → `SharpeRatioData` | `rfr: float=0.0`, `window`, `index` | `date`, `sharpe_ratio` |
| `sortino_ratio` | `SortinoRatioQueryParams` → `SortinoRatioData` | `target_return: float=0.0`, `adjusted: bool=False` | `date`, `sortino_ratio` |
| `omega_ratio` | `OmegaRatioQueryParams` → `OmegaRatioData` | `threshold_start=0.0`, `threshold_end=1.5`, `bins: PositiveInt=50` | `threshold`, `omega` |
| `capm` | `CapmQueryParams` → `CapmData` | — | `market_risk`, `systematic_risk`, `idiosyncratic_risk` |
| `summary` | `SummaryQueryParams` → `SummaryData` | — | `count`, `mean`, `std`, `var`, `min`, `p25`, `p50`, `p75`, `max` |
| `normality` | `NormalityQueryParams` → `NormalityData` | — | Thống kê và p-value của kurtosis, skewness, Jarque-Bera… (10 trường) |
| `unitroot_test` | `UnitRootTestQueryParams` → `UnitRootData` | `fuller_reg: Literal['c','ct','ctt','n']`, `kpss_reg: Literal['c','ct']`, `maxlag: NonNegativeInt \| None` | `adf_*`, `kpss_*` (9 trường) |
| `rolling.{skew,variance,stdev,kurtosis,mean}` | `Rolling*QueryParams` → `Rolling*Data` | `window`, `index` | `date` + chỉ số |
| `factors`, `attribution`, `risk_decomposition` | `FactorRegression…`, `ReturnAttribution…`, `RiskDecomposition…` | `factors_data: list[Data]`, `risk_free_column: str \| None`, `periods: list[str]` | `period`, `factor`, `coefficient`/`contribution`, `share`, `p_value`, `lower_ci`, `upper_ci`, `r_squared` |

Các lớp `*QueryParams` này khai báo thuộc tính lớp `__category__` và `__output_columns__` để mô tả cột đầu ra `[OK]`.

### 3.3 Hợp đồng bảng dữ liệu phía máy khách (AG Grid, `app/model/aggrid.py`)

| Mô hình | Trường |
|---|---|
| `AgGridRowsRequest` | `startRow: int` (R), `endRow: int` (R), `sortModel: list[AgSortColumn]` (O), `filterModel: dict[str, AgFilterModel]` (O), `grouping: AgGroupingModel \| None` (O) |
| `AgSortColumn` | `colId: str`, `sort: SortDirection` (R) |
| `AgFilterModel` | `filterType: FilterType \| None`, `type: str`, `filter: Any`, `filterTo: Any`, `values: list[Any] \| None` |
| `AgGroupingModel` | `rowGroupCols`, `valueCols`, `pivotCols` (`list[dict]`), `pivotMode: bool`, `groupKeys: list[str]` |
| `AgGridRowsResponse[Row]` | `rowData: list[Row]`, `rowCount: int` |

---

## 4. Data Contracts Lỗi (L3)

| Hợp đồng | Nguồn | Nội dung |
|---|---|---|
| `OpenBBErrorResponse` | `app/router.py:38` | `detail: str` (R), `error_kind: str` (R). Gắn vào phản hồi OpenAPI các mã 400, 500, 502 |
| Thân lỗi thực tế | `api/exception_handlers.py:_handle` | `{"detail": <str \| list>}` |
| Mô tả OpenAPI mặc định | `app/router.py:146-167` | 204 `Empty response`; 400 `No Results Found`; 404 `Not found`; 500 `Internal Error`; 502 `Unauthorized` |

Ánh xạ ngoại lệ → HTTP: `OpenBBError` 400; `EmptyDataError` 204 (không thân); `UnauthorizedError` 502; lỗi validate 422 (`detail` là danh sách mục lỗi có `loc` bắt đầu bằng `query`); `Exception` 500 (`Unexpected Error -> <class> -> <msg>`). Xem `03_Core_Business_Logic.md` mục 4.

**[!] Bất nhất hợp đồng lỗi:** schema OpenAPI hứa `{detail, error_kind}` nhưng không có đoạn mã nào đặt `error_kind` (`grep error_kind` chỉ khớp khai báo và `responses`). Máy khách sinh mã từ OpenAPI sẽ kỳ vọng trường không tồn tại. Phản hồi 400 được mô tả là `No Results Found` trong khi `EmptyDataError` trả 204 và `OpenBBError` đại diện mọi lỗi nghiệp vụ. Mã 404 không có mô hình.

---

## 5. Data Contracts Cấu hình (L4)

### 5.1 `UserSettings` (`model/user_settings.py`, kế thừa `Tagged`)

| Trường | Kiểu | R/O |
|---|---|---|
| `credentials` | `Credentials` | O (`default_factory`) |
| `preferences` | `Preferences` | O |
| `defaults` | `Defaults` | O |

### 5.2 `Credentials` (`model/credentials.py`) — mô hình động

Sinh bằng `create_model` từ `CredentialsLoader`: mỗi khóa credential của mọi provider đã cài là một trường `Optional[OBBSecretStr]`, `alias` là tên viết hoa (`FRED_API_KEY`), `description` là tên provider; trường được sắp xếp theo tên. Khóa lạ bổ sung từ tệp cũng thành trường cùng kiểu. Cảnh báo `Skipping '<name>', credential already in use.` khi trùng khóa. `__repr__` che giá trị; `show()` hiện giá trị. Giá trị rỗng/`SecretStr` rỗng coi là chưa đặt (`_is_unset`).

### 5.3 `Preferences` (`model/preferences.py`)

| Trường | Kiểu | Mặc định |
|---|---|---|
| `cache_directory` | `str` | `~/OpenBBUserData/cache` |
| `chart_style` | `Literal['dark','light']` | `dark` |
| `data_directory` | `str` | `~/OpenBBUserData` |
| `export_directory` | `str` | `~/OpenBBUserData/exports` |
| `metadata` | `bool` | `True` |
| `output_type` | `Literal['OBBject','dataframe','polars','numpy','dict','llm']` | `OBBject` |
| `request_timeout` | `PositiveInt` | `60` |
| `show_warnings` | `bool` | `False` |
| `table_style` | `Literal['dark','light']` | `dark` |
| `user_styles_directory` | `str` | `~/OpenBBUserData/styles/user` |

### 5.4 `Defaults` (`model/defaults.py`)

`commands: dict[str, dict[str, Any]]` — giá trị mặc định tham số theo route (khóa dạng `equity.price.historical`). Quy tắc áp dụng trong `commands.py:wrapper`: `provider` bị bỏ; `chart` và `chart_params` ánh xạ riêng; tham số `None` trong `standard_params` được điền; tham số người dùng đã cung cấp không bị ghi đè. Validator `validate_before` chuẩn hóa khóa `routes` cũ và giá trị `show_warnings` dạng chuỗi.

### 5.5 `SystemSettings`, `APISettings`, `PythonSettings`, `CommandContext`

| Mô hình | Trường chính |
|---|---|
| `SystemSettings` (26 trường) | `os`, `python_version`, `platform`, `version`, `core`, đường dẫn tệp cấu hình, nhóm `logging_*` (`logging_frequency: Literal['D','H','M','S']`, `logging_sub_app: Literal['python','api','pro','cli']`, `logging_suppress: bool`), `api_settings`, `python_settings`, `debug_mode`, `test_mode`, `headless`, `allow_mutable_extensions`, `allow_on_command_output`, `charting_extension`, `charting_backend` |
| `APISettings` (`frozen`) | `version="1"`, `title`, `description`, `terms_of_service`, thông tin liên hệ và giấy phép, `servers: list[Servers]`, `cors: Cors`, `custom_headers: dict[str,str] \| None`, thuộc tính tính toán `prefix` |
| `Cors` | `allow_origins`, `allow_methods`, `allow_headers` — mặc định `["*"]` |
| `Servers` | `url: str`, `description: str` |
| `PythonSettings` | `docstring_sections`, `docstring_max_length`, `http: dict \| None`, `uvicorn: dict \| None` |
| `CommandContext` | `user_settings: UserSettings`, `system_settings: SystemSettings` — tiêm vào hàm lệnh, ẩn khỏi schema API |

---

## 6. Data Contracts Tích hợp (L5)

### 6.1 Spec lệnh Platform API / MCP (`app/spec.py`, định nghĩa trùng ở hai gói)

| Mô hình | Trường |
|---|---|
| `_SpecDocumentModel` (11 trường) | `version: int`, `base_url: str`, `api_prefix: str`, `commands: dict[str, _CommandSpecModel]`, `content_sha256: str`, `routers: dict`, `reference: dict`, `generated_at: str \| None`, `generator: str \| None`… |
| `_CommandSpecModel` | `url_path: str`, `method: str`, `parameters: list[_CommandParameterModel]`, `request_body_schema: dict \| None`, `response_schema: dict \| None` |
| `_CommandParameterModel` | `name: str` (R), `type: str`, `is_list: bool`, `required: bool`, `default: Any`, `choices: list[Any]` |

`[!]` Ba lớp trên được sao chép nguyên văn giữa `openbb_platform_api/app/spec.py` và `openbb_mcp_server/app/spec.py`; thay đổi một nơi dễ làm hai hợp đồng lệch nhau.

### 6.2 Widget response (Platform API, `models/response.py`, `models/query.py`) — kế thừa `Data`

| Mô hình | Trường |
|---|---|
| `MetricResponseModel` | `label: str` (R), `value: int \| float \| str` (R), `delta: int \| float \| str \| None` |
| `PdfResponseModel` | `filename`, `content: str \| bytes`, `url`, `data_format: dict` (đều O) |
| `OmniWidgetResponseModel` | `content: Any` (R), `parse_as: str \| None`, `data_format: dict \| None` |
| `OmniWidgetInput` | `prompt: Any \| None` |

### 6.3 Cấu hình MCP (`openbb_mcp_server/models/`)

| Mô hình | Trường chính |
|---|---|
| `MCPSettings` (23 trường) | `api_prefix`, `name`, `description`, `version`, `default_tool_categories`, `allowed_tool_categories`, `enable_tool_discovery`, `list_page_size`, `describe_responses`, `instructions`, `system_prompt_file`, `server_prompts_file`, `default_skills_dir`, `on_duplicate`, `mask_error_details`, `skills_reload`, `skills_providers`, `module_exclusion_map`, `enable_cli_tools`, `uvicorn_config`, `httpx_client_kwargs`, `client_auth: tuple[str,str] \| None`, `server_auth: tuple[str,str] \| None` |
| `MCPConfigModel` (10 trường) | `expose: bool \| None`, `mcp_type: MCPType \| None`, `methods: list[HTTPMethod]`, `prompts: list[PromptConfigModel]`, `exclude_args`, `name`, `tags`, `enable`… — cấu hình qua `openapi_extra["mcp_config"]` của lệnh |
| `PromptConfigModel` | `name`, `description`, `content: str` (R), `arguments: list[ArgumentDefinitionModel]`, `tags` |
| `ArgumentDefinitionModel` | `name`, `type`, `default`, `description` |
| `ToolInfo`, `SubcategoryInfo`, `CategoryInfo` | `name`/`description`; `name`/`tool_count`; `name`/`subcategories`/`total_tools` |
| `PipelineStep` | `id: str \| None`, `tool: str` (R), `arguments: dict`, `inputs: dict[str,str]` |

### 6.4 Siêu dữ liệu OpenAPI mở rộng (`Router.command`, `router.py:101-115`)

| Khóa `openapi_extra` | Nội dung |
|---|---|
| `widget_config` | Cấu hình widget Workspace (ví dụ `{"exclude": True}`) |
| `mcp_config` | `MCPConfigModel` theo lệnh |
| `model` | Tên model chuẩn của lệnh (`EquityHistorical`…) |

---

## 7. Quy tắc validate xuyên suốt

| ID | Quy tắc | Bằng chứng |
|---|---|---|
| VR-01 | `QueryParams` chấp nhận trường thừa (`extra="allow"`); validate kiểu chỉ áp cho trường khai báo. | `query_params.py` |
| VR-02 | Lỗi validate của `QueryParams`/`Data` phải chuyển thành `OpenBBError` có định dạng `[Data Model] <title>` và `[Arg] <loc> -> input: <v> -> <msg>` trên SDK; trên REST thành 422 với `loc` tiền tố `query`. | `decorators.py:63-86`, `exception_handlers.py` |
| VR-03 | `symbol` chuẩn hóa chữ hoa ở các model `Equity*`. | `equity_historical.py:31` |
| VR-04 | Trường có `multiple_items_allowed` cho phép danh sách phân tách dấu phẩy; provider tự tách chuỗi. | `fred/models/series.py` |
| VR-05 | `window` phải thuộc `PositiveInt` và không vượt số quan sát (`validate_window` → `ValueError`). | `quantitative/helpers.py:53` |
| VR-06 | Tham số `ExtraParams` không được provider hỗ trợ bị loại và phát `OpenBBWarning`. | `query.py:filter_extra_params` |
| VR-07 | `Credentials` dùng `OBBSecretStr` để che giá trị khi in/tuần tự hóa. | `credentials.py` |

---

## 8. Rủi ro hợp đồng

| ID | Phát hiện | Mức | Khuyến nghị |
|---|---|---|---|
| DC-01 | `OpenBBErrorResponse.error_kind` được công bố trong OpenAPI nhưng không được điền. | Cao | Điền `error_kind` trong `_handle` hoặc bỏ khỏi mô hình. |
| DC-02 | Schema phụ thuộc tập provider đã cài (`ProviderChoices`, `ExtraParams`). | Trung bình | Công bố schema theo bản dựng môi trường; không dùng schema làm hợp đồng cố định giữa môi trường. |
| DC-03 | `Data` và `QueryParams` dùng `extra="allow"`: lỗi chính tả tên tham số không bị từ chối, chỉ bị bỏ qua hoặc chuyển tiếp. | Trung bình | Cân nhắc cảnh báo cho khóa không khai báo ở tầng REST. |
| DC-04 | Mô hình spec trùng lặp giữa `platform_api` và `mcp_server`. | Thấp | Chuyển về một gói dùng chung. |
| DC-05 | Mã 400 mô tả `No Results Found` nhưng ngữ nghĩa thực tế là lỗi nghiệp vụ chung; dữ liệu rỗng trả 204. | Thấp | Chỉnh mô tả OpenAPI. |
| DC-06 | Hợp đồng chi tiết của 2.197 lớp `QueryParams`/`Data` trong provider/extension chỉ tổng hợp theo số lượng; từng lớp nằm ngoài phạm vi. | — | `[OPEN]` Dùng `GET /coverage/command_model` để xuất schema theo môi trường. |
