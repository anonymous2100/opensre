# Gitee MCP 集成方案（离线版）

> 适用代码：当前工作树 `dev_lh2`，`pyproject.toml` 版本 `0.1`。  
> 文档性质：离线方案文档，不依赖 Mintlify 或网络。  
> 实现状态：已在当前分支 `dev_lh2` 落地；本文保留完整代码、接线说明和测试计划，便于 review 与后续维护。  
> 代码校验：`integrations/gitee` 与相关接线已通过 `ruff check`、`mypy` 和针对性 `pytest`。  
> 目标：让 OpenSRE 通过 Gitee 官方 MCP Server 读取 Gitee 托管的项目源码，完成目录浏览、文件读取、内容搜索和 ref 对比，并把结果纳入 evidence 链。

---

## 目录

- [1. 背景与目标](#1-背景与目标)
- [2. 设计决策](#2-设计决策)
- [3. 代码结构](#3-代码结构)
- [4. 新增文件代码](#4-新增文件代码)
- [5. 已有文件接线](#5-已有文件接线)
- [6. 环境变量与配置](#6-环境变量与配置)
- [7. 使用流程](#7-使用流程)
- [8. 测试计划](#8-测试计划)
- [9. 限制与后续演进](#9-限制与后续演进)
- [10. 参考](#10-参考)

---

## 1. 背景与目标

OpenSRE 现有的代码仓库读取工具面向 GitHub / GitLab / Bitbucket。Gitee 没有内置集成，但 Gitee 官方提供了 MCP Server（`oschina/mcp-gitee`），支持远程 MCP 和本地 stdio 两种接入方式。

本方案选择**适配官方 mcp-gitee**，而不是直接包 Gitee OpenAPI：

- 远程端点：`https://api.gitee.com/mcp`，请求头 `Authorization: Bearer <personal access token>`；
- 本地 stdio：`npx -y @gitee/mcp-gitee@latest` 或本机 `mcp-gitee`，环境变量 `GITEE_ACCESS_TOKEN`、`GITEE_API_BASE`；
- 服务端已知只读工具：`list_user_repos`、`get_file_content`、`search_files_by_content`、`compare_branches_tags`、`get_diff_files`、`list_repo_pulls`、`list_repo_issues`、`get_repo_issue_detail`、`list_comments`。

本方案的目标不是复制 GitHub 集成的全部能力，而是覆盖故障调查中的**只读代码分析**：

1. 解析或列出 `owner/repo`；
2. 按目录浏览仓库；
3. 读取指定 branch / tag / commit 上的文件；
4. 按内容搜索代码；
5. 比较两个 ref 的差异；
6. 把读取到的文件、目录和搜索写入 evidence，最终回答可引用。

---

## 2. 设计决策

| 决策 | 选择 | 理由 |
| --- | --- | --- |
| 集成名 | `gitee_mcp`，别名 `gitee` | 与 `sentry_mcp` / `posthog_mcp` 一致；未来可再拆出 REST 版 `gitee` |
| 默认传输 | `streamable-http` → `https://api.gitee.com/mcp` | 官方推荐的免安装远程方式 |
| 兜底传输 | `stdio` → `npx -y @gitee/mcp-gitee@latest` | 远程不可用、版本锁定或内网环境 |
| 工具面 | 通用桥 + 只读便捷工具 | 服务端增删工具时通用桥仍可用；便捷工具让模型更容易选对 |
| 默认权限 | 只读 allowlist | Token 只需仓库读权限；写工具必须显式加入白名单 |
| 通用桥审批 | `requires_approval=True` | 白名单被放宽后，通用桥可能调用写工具，必须人工确认 |
| 输出控制 | `get_gitee_file` 的 `max_chars` | Gitee 文件可能很大，先搜索再读，避免上下文爆炸 |
| Evidence | 每个只读便捷工具带 `evidence_mapper` | 最终回答能引用具体文件、目录、搜索和 ref 对比 |

---

## 3. 代码结构

```text
config/constants/gitee.py
integrations/gitee/__init__.py
integrations/gitee/setup.py
integrations/gitee/verifier.py
integrations/gitee/tools/__init__.py
integrations/gitee/tools/_shared.py
integrations/gitee/tools/gitee_mcp_tool/__init__.py
integrations/gitee/tools/gitee_code_tool/__init__.py
```

`integrations/gitee/tools/_shared.py` 以 `_` 开头，工具发现扫描会跳过它，但它可以被两个工具子包正常 import。

---

## 4. 新增文件代码

### 4.1 `config/constants/gitee.py`

```python
"""Gitee MCP environment variable names."""

from __future__ import annotations

GITEE_ACCESS_TOKEN_ENV = "GITEE_ACCESS_TOKEN"
GITEE_MCP_URL_ENV = "GITEE_MCP_URL"
GITEE_MCP_AUTH_TOKEN_ENV = "GITEE_MCP_AUTH_TOKEN"
GITEE_MCP_MODE_ENV = "GITEE_MCP_MODE"
GITEE_MCP_COMMAND_ENV = "GITEE_MCP_COMMAND"
GITEE_MCP_ARGS_ENV = "GITEE_MCP_ARGS"
GITEE_MCP_TIMEOUT_SECONDS_ENV = "GITEE_MCP_TIMEOUT_SECONDS"
GITEE_MCP_ENABLED_TOOLS_ENV = "GITEE_MCP_ENABLED_TOOLS"

__all__ = [
    "GITEE_ACCESS_TOKEN_ENV",
    "GITEE_MCP_ARGS_ENV",
    "GITEE_MCP_AUTH_TOKEN_ENV",
    "GITEE_MCP_COMMAND_ENV",
    "GITEE_MCP_ENABLED_TOOLS_ENV",
    "GITEE_MCP_MODE_ENV",
    "GITEE_MCP_TIMEOUT_SECONDS_ENV",
    "GITEE_MCP_URL_ENV",
]
```
### 4.2 `integrations/gitee/__init__.py`

负责配置模型、环境变量解析、MCP 会话、工具发现/调用、连通性校验和存储记录分类。

```python
"""Gitee MCP integration.

The official Gitee MCP server (``https://api.gitee.com/mcp``) exposes
repository listing, file contents, content search, and ref comparison. This
package owns config normalization, MCP transport, validation, and the shared
payload helpers used by the agent tools.
"""

from __future__ import annotations

import logging
import os
from collections.abc import Mapping
from dataclasses import dataclass
from http import HTTPStatus
from typing import Any, cast

import httpx
from pydantic import Field, field_validator, model_validator
from typing_extensions import TypedDict

from config.constants.gitee import (
    GITEE_ACCESS_TOKEN_ENV,
    GITEE_MCP_ARGS_ENV,
    GITEE_MCP_AUTH_TOKEN_ENV,
    GITEE_MCP_COMMAND_ENV,
    GITEE_MCP_ENABLED_TOOLS_ENV,
    GITEE_MCP_MODE_ENV,
    GITEE_MCP_TIMEOUT_SECONDS_ENV,
    GITEE_MCP_URL_ENV,
)
from config.strict_config import StrictConfigModel
from integrations._validation_helpers import report_classify_failure, report_validation_failure
from integrations.mcp_client import (
    McpSessionOptions,
    call_mcp_tool,
    list_mcp_tools,
    root_cause_message,
)
from integrations.mcp_transport import McpTransportMode

logger = logging.getLogger(__name__)

DEFAULT_GITEE_MCP_URL = "https://api.gitee.com/mcp"
DEFAULT_GITEE_MCP_MODE: McpTransportMode = McpTransportMode.STREAMABLE_HTTP
GITEE_API_BASE = "https://gitee.com/api/v5"

#: Read-only tools the code-analysis path needs. Keeping this allowlist on by
#: default means a Gitee token with repository read scope can never create,
#: merge, comment, or push, even through the generic bridge.
DEFAULT_GITEE_READ_TOOLS: tuple[str, ...] = (
    "list_user_repos",
    "get_file_content",
    "search_files_by_content",
    "compare_branches_tags",
    "get_diff_files",
    "list_repo_pulls",
    "list_repo_issues",
    "get_repo_issue_detail",
    "list_comments",
)
_EXPECTED_CODE_TOOLS = frozenset({"get_file_content", "search_files_by_content"})


class GiteeMCPToolDescriptor(TypedDict):
    """A tool exposed by the Gitee MCP server."""

    name: str
    description: str
    input_schema: object | None


class GiteeMCPContentItem(TypedDict, total=False):
    """Normalized content item returned by an MCP tool call."""

    type: str
    text: str
    uri: str
    mime_type: str


class GiteeMCPToolCallResult(TypedDict, total=False):
    """Normalized response from a Gitee MCP tool call."""

    is_error: bool
    text: str
    content: list[GiteeMCPContentItem]
    structured_content: object | None
    tool: str
    arguments: dict[str, object]


class GiteeMCPConfig(StrictConfigModel):
    """Normalized Gitee MCP connection settings."""

    url: str = DEFAULT_GITEE_MCP_URL
    mode: McpTransportMode = DEFAULT_GITEE_MCP_MODE
    auth_token: str = ""
    command: str = ""
    args: tuple[str, ...] = ()
    timeout_seconds: float = Field(default=30.0, gt=0)
    enabled_tools: tuple[str, ...] = DEFAULT_GITEE_READ_TOOLS
    integration_id: str = ""

    @field_validator("url", mode="before")
    @classmethod
    def _normalize_url(cls, value: object) -> str:
        normalized = str(value or "").strip()
        return normalized.rstrip("/") if normalized else ""

    @field_validator("mode", mode="before")
    @classmethod
    def _normalize_mode(cls, value: object) -> str:
        normalized = str(value or DEFAULT_GITEE_MCP_MODE).strip().lower()
        return normalized or DEFAULT_GITEE_MCP_MODE

    @field_validator("auth_token", mode="before")
    @classmethod
    def _normalize_auth_token(cls, value: object) -> str:
        token = str(value or "").strip()
        if token.lower().startswith("bearer "):
            token = token.split(None, 1)[1].strip()
        return token

    @field_validator("command", mode="before")
    @classmethod
    def _normalize_command(cls, value: object) -> str:
        return str(value or "").strip()

    @field_validator("args", mode="before")
    @classmethod
    def _normalize_args(cls, value: object) -> tuple[str, ...]:
        if value is None or not isinstance(value, (list, tuple, set)):
            return ()
        return tuple(str(arg).strip() for arg in value if str(arg).strip())

    @field_validator("enabled_tools", mode="before")
    @classmethod
    def _normalize_enabled_tools(cls, value: object) -> tuple[str, ...]:
        if value is None:
            return DEFAULT_GITEE_READ_TOOLS
        if isinstance(value, str):
            candidates = value.replace(",", " ").split()
        elif isinstance(value, (list, tuple, set)):
            candidates = [str(item) for item in value]
        else:
            return DEFAULT_GITEE_READ_TOOLS
        tools = tuple(item.strip() for item in candidates if item.strip())
        # An empty allowlist would mean "all tools" to the server; keep the
        # read-only default instead of silently widening the surface.
        return tools or DEFAULT_GITEE_READ_TOOLS

    @model_validator(mode="after")
    def _validate_transport_requirements(self) -> GiteeMCPConfig:
        if self.mode == McpTransportMode.STDIO and not self.command:
            raise ValueError("Gitee MCP mode 'stdio' requires a non-empty command.")
        if self.mode != McpTransportMode.STDIO and not self.url:
            raise ValueError(f"Gitee MCP mode '{self.mode}' requires a non-empty url.")
        return self

    @property
    def is_configured(self) -> bool:
        if self.mode == McpTransportMode.STDIO:
            return bool(self.command)
        return bool(self.url)

    @property
    def request_headers(self) -> dict[str, str]:
        """Headers for the remote MCP endpoint, including the read-only allowlist."""
        headers: dict[str, str] = {}
        if self.auth_token:
            headers["Authorization"] = f"Bearer {self.auth_token}"
        if self.enabled_tools and self.mode != McpTransportMode.STDIO:
            headers["X-MCP-Enabled-Tools"] = ",".join(self.enabled_tools)
        return headers


@dataclass(frozen=True)
class GiteeMCPValidationResult:
    """Result of validating a Gitee MCP connection."""

    ok: bool
    detail: str
    tool_names: tuple[str, ...] = ()


def build_gitee_mcp_config(raw: Mapping[str, object] | None) -> GiteeMCPConfig:
    """Build a normalized Gitee MCP config from env/store data."""
    payload = dict(raw or {})
    allowed = set(GiteeMCPConfig.model_fields)
    sanitized = {key: value for key, value in payload.items() if key in allowed}
    mode = str(sanitized.get("mode") or DEFAULT_GITEE_MCP_MODE).strip().lower()
    if mode != McpTransportMode.STDIO and not str(sanitized.get("url") or "").strip():
        # The remote endpoint is the default; a record that omits the URL (or
        # carries "" from an old setup form) should still resolve to it.
        sanitized["url"] = DEFAULT_GITEE_MCP_URL
    return GiteeMCPConfig.model_validate(sanitized)


def _timeout_from_env() -> float:
    raw = os.getenv(GITEE_MCP_TIMEOUT_SECONDS_ENV, "").strip()
    if not raw:
        return 30.0
    try:
        return max(1.0, float(raw))
    except ValueError:
        return 30.0


def gitee_mcp_config_from_env() -> GiteeMCPConfig | None:
    """Load a Gitee MCP config from environment variables."""
    mode = os.getenv(GITEE_MCP_MODE_ENV, DEFAULT_GITEE_MCP_MODE).strip().lower()
    mode = mode or DEFAULT_GITEE_MCP_MODE
    url = os.getenv(GITEE_MCP_URL_ENV, "").strip()
    command = os.getenv(GITEE_MCP_COMMAND_ENV, "").strip()
    token = (
        os.getenv(GITEE_MCP_AUTH_TOKEN_ENV, "").strip()
        or os.getenv(GITEE_ACCESS_TOKEN_ENV, "").strip()
    )
    args = tuple(part for part in os.getenv(GITEE_MCP_ARGS_ENV, "").split() if part)

    if mode == McpTransportMode.STDIO:
        if not command:
            return None
    else:
        if not token:
            return None
        if not url:
            url = DEFAULT_GITEE_MCP_URL

    enabled_raw = os.getenv(GITEE_MCP_ENABLED_TOOLS_ENV, "").strip()
    return build_gitee_mcp_config(
        {
            "url": url,
            "mode": mode,
            "command": command,
            "args": args,
            "auth_token": token,
            "timeout_seconds": _timeout_from_env(),
            "enabled_tools": enabled_raw or DEFAULT_GITEE_READ_TOOLS,
        }
    )


def gitee_mcp_runtime_unavailable_reason(config: GiteeMCPConfig) -> str | None:
    """Return a setup error when the config cannot be used."""
    if not config.is_configured:
        return "Gitee MCP is not configured: provide a URL (HTTP/SSE) or command (stdio)."
    if config.mode != McpTransportMode.STDIO and not config.auth_token:
        return (
            "Gitee MCP requires a personal access token. Create one at "
            "https://gitee.com/profile/personal_access_tokens with repository read "
            "scope and set GITEE_MCP_AUTH_TOKEN."
        )
    return None


def _session_options(config: GiteeMCPConfig) -> McpSessionOptions:
    stdio_env: dict[str, str] = {"GITEE_API_BASE": GITEE_API_BASE}
    if config.auth_token:
        stdio_env["GITEE_ACCESS_TOKEN"] = config.auth_token
    if config.enabled_tools:
        stdio_env["ENABLED_TOOLSETS"] = ",".join(config.enabled_tools)
    return {
        "session_url": config.url,
        "stdio_env": stdio_env,
        "integration_name": "Gitee",
        "config_env_name": "GITEE_MCP",
        "sse_url_hint": DEFAULT_GITEE_MCP_URL,
        "streamable_url_hint": DEFAULT_GITEE_MCP_URL,
    }


def describe_gitee_mcp_error(err: BaseException, config: GiteeMCPConfig) -> str:
    """Render a human-readable error with a setup hint when useful."""
    detail = root_cause_message(err, timeout_message="Gitee MCP tool call timed out")
    hints: list[str] = []
    if isinstance(err, httpx.HTTPStatusError) and err.response.status_code in (
        HTTPStatus.UNAUTHORIZED,
        HTTPStatus.FORBIDDEN,
    ):
        hints.append(
            "Authentication failed. Check GITEE_MCP_AUTH_TOKEN is a valid Gitee "
            "personal access token with repository read scope."
        )
    elif config.mode != McpTransportMode.STDIO and not config.auth_token:
        hints.append("No auth token configured. Set GITEE_MCP_AUTH_TOKEN.")
    if "timed out" in detail.lower():
        hints.append(
            f"The tool did not return within {config.timeout_seconds:.1f}s. "
            "Raise GITEE_MCP_TIMEOUT_SECONDS if the tool is expected to be slow."
        )
    return f"{detail} Hint: {' '.join(hints)}" if hints else detail


def list_gitee_mcp_tools(config: GiteeMCPConfig) -> list[GiteeMCPToolDescriptor]:
    """List available tools from the Gitee MCP server."""
    return [
        cast(
            GiteeMCPToolDescriptor,
            {
                "name": tool.name,
                "description": tool.description or "",
                "input_schema": tool.input_schema,
            },
        )
        for tool in list_mcp_tools(config, **_session_options(config))
    ]


def call_gitee_mcp_tool(
    config: GiteeMCPConfig,
    tool_name: str,
    arguments: dict[str, object] | None = None,
) -> GiteeMCPToolCallResult:
    """Call a Gitee MCP tool and normalize the result."""
    return cast(
        GiteeMCPToolCallResult,
        call_mcp_tool(
            config,
            tool_name,
            arguments,
            timeout_call=True,
            **_session_options(config),
        ),
    )


def validate_gitee_mcp_config(config: GiteeMCPConfig) -> GiteeMCPValidationResult:
    """Validate Gitee MCP connectivity and the read-only tool surface."""
    runtime_error = gitee_mcp_runtime_unavailable_reason(config)
    if runtime_error is not None:
        return GiteeMCPValidationResult(
            ok=False,
            detail=f"Gitee MCP validation failed: {runtime_error}",
        )

    try:
        tool_names = tuple(sorted({tool["name"] for tool in list_gitee_mcp_tools(config)}))
        if not tool_names:
            return GiteeMCPValidationResult(
                ok=False,
                detail=f"Gitee MCP connected via {config.mode} but exposed no tools.",
            )
        missing = sorted(_EXPECTED_CODE_TOOLS - set(tool_names))
        if missing:
            return GiteeMCPValidationResult(
                ok=False,
                detail=(
                    "Gitee MCP connected but is missing required read tools: "
                    f"{', '.join(missing)}. Check GITEE_MCP_ENABLED_TOOLS and the "
                    "token scopes."
                ),
                tool_names=tool_names,
            )
        endpoint = config.command if config.mode == McpTransportMode.STDIO else config.url
        return GiteeMCPValidationResult(
            ok=True,
            detail=(
                f"Gitee MCP connected via {config.mode} ({endpoint}); "
                f"discovered {len(tool_names)} tool(s)."
            ),
            tool_names=tool_names,
        )
    except Exception as err:
        report_validation_failure(
            err,
            logger=logger,
            integration="gitee_mcp",
            method="validate_gitee_mcp_config",
        )
        return GiteeMCPValidationResult(
            ok=False,
            detail=f"Gitee MCP validation failed: {describe_gitee_mcp_error(err, config)}",
        )


def classify(credentials: dict[str, Any], record_id: str) -> tuple[GiteeMCPConfig | None, str | None]:
    """Classify a stored Gitee MCP record into a runtime config."""
    try:
        config = build_gitee_mcp_config(
            {
                "url": credentials.get("url", ""),
                "mode": credentials.get("mode", DEFAULT_GITEE_MCP_MODE),
                "command": credentials.get("command", ""),
                "args": credentials.get("args", []),
                "auth_token": credentials.get("auth_token", ""),
                "enabled_tools": credentials.get("enabled_tools", DEFAULT_GITEE_READ_TOOLS),
                "timeout_seconds": credentials.get("timeout_seconds", 30.0),
                "integration_id": record_id,
            }
        )
    except Exception as exc:
        report_classify_failure(exc, logger=logger, integration="gitee_mcp", record_id=record_id)
        return None, None
    if config.is_configured and (
        config.mode == McpTransportMode.STDIO or config.auth_token
    ):
        return config, "gitee_mcp"
    return None, None


def normalize_gitee_tool_result(result: GiteeMCPToolCallResult) -> dict[str, object]:
    """Convert an MCP tool result into the standard integration payload shape."""
    if result.get("is_error"):
        return {
            "source": "gitee_mcp",
            "available": False,
            "error": str(result.get("text") or "Gitee MCP tool call failed."),
            "tool": result.get("tool"),
            "arguments": result.get("arguments", {}),
        }
    return {
        "source": "gitee_mcp",
        "available": True,
        "tool": result.get("tool"),
        "arguments": result.get("arguments", {}),
        "text": result.get("text", ""),
        "structured_content": result.get("structured_content"),
        "content": result.get("content", []),
    }


__all__ = [
    "DEFAULT_GITEE_MCP_MODE",
    "DEFAULT_GITEE_MCP_URL",
    "DEFAULT_GITEE_READ_TOOLS",
    "GITEE_API_BASE",
    "GiteeMCPConfig",
    "GiteeMCPContentItem",
    "GiteeMCPToolCallResult",
    "GiteeMCPToolDescriptor",
    "GiteeMCPValidationResult",
    "build_gitee_mcp_config",
    "call_gitee_mcp_tool",
    "classify",
    "describe_gitee_mcp_error",
    "gitee_mcp_config_from_env",
    "gitee_mcp_runtime_unavailable_reason",
    "list_gitee_mcp_tools",
    "normalize_gitee_tool_result",
    "validate_gitee_mcp_config",
]
```
### 4.3 `integrations/gitee/verifier.py`

复用仓库的 `register_validation_verifier`，让 `opensre integrations verify gitee` 与 shell 内的 `/integrations verify` 走同一条校验路径。

```python
"""Gitee MCP integration verifier."""

from __future__ import annotations

from integrations.gitee import build_gitee_mcp_config, validate_gitee_mcp_config
from integrations.verification import register_validation_verifier

verify_gitee_mcp = register_validation_verifier(
    "gitee_mcp",
    build_config=build_gitee_mcp_config,
    validate_config=validate_gitee_mcp_config,
)
```
### 4.4 `integrations/gitee/setup.py`

只提示用户真正需要填的两项：MCP URL 和 token。传输模式、命令、参数、超时和白名单都放在环境变量里，避免 setup 流程持久化没人选择的值。

```python
"""What the Gitee MCP server needs before it is considered configured.

Transport details (``mode``/``command``/``args``) are environment-only, like
the Sentry MCP setup: a user is prompted for the endpoint and the token, and
the model defaults already cover the remote Streamable HTTP case.
"""

from __future__ import annotations

from config.constants.gitee import GITEE_MCP_AUTH_TOKEN_ENV, GITEE_MCP_URL_ENV
from integrations.gitee import DEFAULT_GITEE_MCP_URL
from integrations.gitee.verifier import verify_gitee_mcp
from integrations.setup_flow import IntegrationSetupSpec, SetupField

URL_FIELD = "url"
AUTH_TOKEN_FIELD = "auth_token"

GITEE_MCP_SETUP = IntegrationSetupSpec(
    service="gitee_mcp",
    fields=(
        SetupField(
            name=URL_FIELD,
            label="Gitee MCP URL",
            env_var=GITEE_MCP_URL_ENV,
            default=DEFAULT_GITEE_MCP_URL,
        ),
        SetupField(
            name=AUTH_TOKEN_FIELD,
            label="Gitee personal access token",
            prompt="Gitee personal access token (repository read scope)",
            env_var=GITEE_MCP_AUTH_TOKEN_ENV,
            secret=True,
        ),
    ),
    verify=verify_gitee_mcp,
)

__all__ = [
    "AUTH_TOKEN_FIELD",
    "GITEE_MCP_SETUP",
    "URL_FIELD",
]
```
### 4.5 `integrations/gitee/tools/__init__.py`

```python
"""Gitee MCP-backed agent tools."""
```
### 4.6 `integrations/gitee/tools/_shared.py`

两个工具子包共用的连接解析、调用、payload 解码、文件视图和 evidence 记录。

```python
"""Shared helpers for the Gitee MCP agent tools.

The discovery scanner skips modules whose name starts with ``_``, so this file
is importable by both tool packages without registering a tool of its own.
"""

from __future__ import annotations

import base64
import contextlib
import json
from typing import Any

from core.domain.types.evidence import record_evidence_entry, unique_evidence_source
from core.tool import report_run_error
from core.tool_framework.utils import first_list, first_string, unavailable_response
from integrations.gitee import (
    DEFAULT_GITEE_MCP_MODE,
    DEFAULT_GITEE_READ_TOOLS,
    GiteeMCPConfig,
    build_gitee_mcp_config,
    call_gitee_mcp_tool,
    describe_gitee_mcp_error,
    gitee_mcp_config_from_env,
    gitee_mcp_runtime_unavailable_reason,
    normalize_gitee_tool_result,
)

COMPONENT = "integrations.gitee.tools._shared"

#: Hidden connection parameters injected by ``extract_params``. Keeping them in
#: every schema lets the tool decorator round-trip stored credentials.
AUTH_PROPERTIES: dict[str, dict[str, object]] = {
    "gitee_url": {"type": "string"},
    "gitee_mode": {"type": "string"},
    "gitee_token": {"type": "string"},
    "gitee_command": {"type": "string"},
    "gitee_args": {"type": "array", "items": {"type": "string"}},
}


def available(sources: dict[str, dict]) -> bool:
    """Whether the verified Gitee MCP connection is present for this turn."""
    return bool(sources.get("gitee_mcp", {}).get("connection_verified"))


def extract_params(sources: dict[str, dict]) -> dict[str, object]:
    """Inject stored Gitee MCP connection settings into a tool call."""
    source = sources.get("gitee_mcp", {})
    if not source:
        return {}
    return {
        "gitee_url": first_string(source, "gitee_url", "url"),
        "gitee_mode": first_string(source, "gitee_mode", "mode"),
        "gitee_token": first_string(source, "gitee_token", "auth_token"),
        "gitee_command": first_string(source, "gitee_command", "command"),
        "gitee_args": first_list(source, "gitee_args", "args"),
    }


def resolve_config(
    url: str | None = None,
    mode: str | None = None,
    token: str | None = None,
    command: str | None = None,
    args: list[str] | None = None,
) -> GiteeMCPConfig | None:
    """Resolve a config from explicit tool arguments or the environment."""
    env_config = gitee_mcp_config_from_env()
    if not any((url, mode, token, command, args)):
        return env_config
    inferred_mode = (
        mode
        or ("stdio" if command else "")
        or ("streamable-http" if url else "")
        or (str(env_config.mode) if env_config else "")
        or str(DEFAULT_GITEE_MCP_MODE)
    )
    return build_gitee_mcp_config(
        {
            "url": url or (env_config.url if env_config else ""),
            "mode": inferred_mode,
            "auth_token": token or (env_config.auth_token if env_config else ""),
            "command": command or (env_config.command if env_config else ""),
            "args": args or (list(env_config.args) if env_config else []),
            "enabled_tools": (
                list(env_config.enabled_tools) if env_config else list(DEFAULT_GITEE_READ_TOOLS)
            ),
            "timeout_seconds": env_config.timeout_seconds if env_config else 30.0,
        }
    )


def config_from_args(
    url: str | None,
    mode: str | None,
    token: str | None,
    command: str | None,
    args: list[str] | None,
) -> tuple[GiteeMCPConfig | None, dict[str, object] | None]:
    """Return ``(config, error_payload)`` for a tool invocation."""
    config = resolve_config(url, mode, token, command, args)
    if config is None:
        return None, unavailable_response(
            "gitee_mcp",
            "Gitee MCP integration is not configured. Run: opensre integrations setup gitee",
        )
    runtime_error = gitee_mcp_runtime_unavailable_reason(config)
    if runtime_error is not None:
        return None, unavailable_response("gitee_mcp", runtime_error)
    return config, None


def dispatch(
    config: GiteeMCPConfig,
    tool_name: str,
    arguments: dict[str, object],
) -> tuple[dict[str, object] | None, dict[str, object] | None]:
    """Call one MCP tool, returning ``(payload, error_payload)``."""
    try:
        result = call_gitee_mcp_tool(config, tool_name, arguments)
    except Exception as err:
        report_run_error(
            err,
            tool_name=tool_name,
            source="gitee_mcp",
            component=COMPONENT,
            method="call_gitee_mcp_tool",
            extras={"mcp_tool": tool_name, "transport": config.mode},
        )
        return None, unavailable_response(
            "gitee_mcp",
            describe_gitee_mcp_error(err, config),
            tool_name=tool_name,
            arguments=arguments,
        )

    normalized = normalize_gitee_tool_result(result)
    if not normalized.get("available"):
        return None, normalized
    return normalized, None


def decode(payload: dict[str, object]) -> object:
    """Return structured content, or JSON-decode the text payload."""
    structured = payload.get("structured_content")
    if structured is not None:
        return structured
    text = str(payload.get("text") or "").strip()
    if not text:
        return ""
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        return text


def truncate(text: str, limit: int) -> tuple[str, bool]:
    """Trim a string to ``limit`` characters and report whether it was cut."""
    if limit <= 0 or len(text) <= limit:
        return text, False
    return text[:limit], True


def file_view(decoded: object, *, max_chars: int) -> dict[str, object]:
    """Normalize a Gitee contents response into a directory or file view."""
    if isinstance(decoded, list):
        return {"kind": "directory", "count": len(decoded), "entries": decoded}
    if isinstance(decoded, dict):
        content = decoded.get("content")
        if isinstance(content, str) and str(decoded.get("encoding") or "").lower() == "base64":
            with contextlib.suppress(ValueError, TypeError):
                content = base64.b64decode(content).decode("utf-8", errors="replace")
        if isinstance(content, str):
            content, truncated = truncate(content, max_chars)
            return {
                "kind": str(decoded.get("type") or "file"),
                "file": decoded,
                "content": content,
                "truncated": truncated,
            }
        return {"kind": str(decoded.get("type") or "file"), "file": decoded, "content": content}
    if isinstance(decoded, str):
        content, truncated = truncate(decoded, max_chars)
        return {"kind": "file", "content": content, "truncated": truncated}
    return {"kind": "file", "content": decoded, "truncated": False}


def record(evidence: dict[str, Any], *, base: str, label: str, summary: str) -> None:
    """Record one citeable evidence entry, keeping repeated calls distinct."""
    record_evidence_entry(
        evidence,
        source=unique_evidence_source(evidence, base),
        label=label,
        summary=summary,
    )
```
### 4.7 `integrations/gitee/tools/gitee_mcp_tool/__init__.py`

通用 MCP 桥：`list_gitee_tools` 发现服务端工具，`call_gitee_tool` 按名称调用。它保证服务端新增或重命名工具时集成不会整体失效。

```python
"""Gitee MCP bridge tools.

A discovery tool plus a named-call bridge keeps the integration working when
the Gitee MCP server adds, renames, or retires individual tools. The read-only
convenience tools live in ``gitee_code_tool``.
"""

from __future__ import annotations

from typing import Any

from core.domain.types.tools import ToolSurface
from core.tool import SideEffectLevel, report_run_error
from core.tool_framework import tool
from core.tool_framework.utils import build_mcp_tool_listing, unavailable_response
from integrations.gitee import describe_gitee_mcp_error, list_gitee_mcp_tools
from integrations.gitee.tools._shared import (
    AUTH_PROPERTIES,
    available,
    config_from_args,
    dispatch,
    extract_params,
    record,
)

_COMPONENT = "integrations.gitee.tools.gitee_mcp_tool"


def _map_call_gitee_tool(
    evidence: dict[str, Any], output: dict[str, Any], _input: dict[str, Any]
) -> None:
    if not output.get("available"):
        return
    record(
        evidence,
        base="call_gitee_tool",
        label="Gitee MCP Tool",
        summary=str(output.get("tool") or "Gitee MCP"),
    )


@tool(
    name="list_gitee_tools",
    source="gitee_mcp",
    side_effect_level=SideEffectLevel.READ_ONLY,
    concurrency_safe=True,
    description=(
        "List the tools exposed by the configured Gitee MCP server. Returns a "
        "compact listing (names and short descriptions, no schemas) by default. "
        "Use name_filter to narrow the list, and include_schema=true on a "
        "narrowed list to fetch the input schema of the tool you intend to call."
    ),
    use_cases=[
        "Discovering which Gitee MCP tools are available before calling one",
        "Finding the right tool for a task with name_filter (e.g. 'file content search')",
        "Fetching the input schema of a specific tool before calling it",
    ],
    surfaces=(ToolSurface.CHAT,),
    input_schema={
        "type": "object",
        "properties": {
            "name_filter": {
                "type": "string",
                "description": (
                    "Optional space- or comma-separated terms; tools whose name or "
                    "description contains any term are returned."
                ),
            },
            "include_schema": {
                "type": "boolean",
                "description": (
                    "Include each tool's full input_schema. Only honored when the "
                    "filtered result set is small; narrow with name_filter first."
                ),
            },
            **AUTH_PROPERTIES,
        },
        "required": [],
    },
    is_available=available,
    extract_params=extract_params,
)
def list_gitee_tools(
    name_filter: str | None = None,
    include_schema: bool = False,
    gitee_url: str | None = None,
    gitee_mode: str | None = None,
    gitee_token: str | None = None,
    gitee_command: str | None = None,
    gitee_args: list[str] | None = None,
    **_kwargs: object,
) -> dict[str, object]:
    """List tools available from the configured Gitee MCP server."""
    config, error = config_from_args(
        gitee_url, gitee_mode, gitee_token, gitee_command, gitee_args
    )
    if error is not None:
        payload = dict(error)
        payload["tools"] = []
        return payload
    assert config is not None

    try:
        tools = list_gitee_mcp_tools(config)
    except Exception as err:
        report_run_error(
            err,
            tool_name="list_gitee_tools",
            source="gitee_mcp",
            component=_COMPONENT,
            method="list_gitee_mcp_tools",
            extras={"transport": config.mode},
        )
        payload = unavailable_response("gitee_mcp", describe_gitee_mcp_error(err, config))
        payload["tools"] = []
        return payload

    listing = build_mcp_tool_listing(
        [dict(descriptor) for descriptor in tools],
        name_filter=(name_filter or "").strip() or None,
        include_schema=bool(include_schema),
        filter_example="file content search compare",
    )
    return {
        "source": "gitee_mcp",
        "available": True,
        "transport": config.mode,
        "endpoint": config.command if config.mode == "stdio" else config.url,
        **listing,
    }


@tool(
    name="call_gitee_tool",
    source="gitee_mcp",
    side_effect_level=SideEffectLevel.EXTERNAL,
    requires_approval=True,
    approval_reason="Calls a named Gitee MCP tool, which may change the repository.",
    description=(
        "Call a named tool exposed by the configured Gitee MCP server. Use this "
        "for tools the read-only convenience surface does not cover. The default "
        "GITEE_MCP_ENABLED_TOOLS allowlist keeps the server read-only; adding a "
        "write tool to that list changes that, so this bridge asks for approval "
        "on every call."
    ),
    use_cases=[
        "Calling a Gitee MCP tool not covered by the read-only convenience tools",
        "Inspecting an issue, pull request, or release during an investigation",
        "Running a vendor tool whose name or schema was discovered with list_gitee_tools",
    ],
    requires=["tool_name"],
    surfaces=(ToolSurface.CHAT,),
    input_schema={
        "type": "object",
        "properties": {
            "tool_name": {"type": "string"},
            "arguments": {"type": "object"},
            **AUTH_PROPERTIES,
        },
        "required": ["tool_name"],
    },
    is_available=available,
    extract_params=extract_params,
    evidence_mapper=_map_call_gitee_tool,
)
def call_gitee_tool(
    tool_name: str | None = None,
    arguments: dict[str, object] | None = None,
    gitee_url: str | None = None,
    gitee_mode: str | None = None,
    gitee_token: str | None = None,
    gitee_command: str | None = None,
    gitee_args: list[str] | None = None,
    **_kwargs: object,
) -> dict[str, object]:
    """Call a specific Gitee MCP tool by name."""
    normalized_name = (tool_name or "").strip()
    call_arguments = arguments or {}
    if not normalized_name:
        return unavailable_response(
            "gitee_mcp",
            "tool_name is required to call a Gitee MCP tool.",
            arguments=call_arguments,
        )

    config, error = config_from_args(
        gitee_url, gitee_mode, gitee_token, gitee_command, gitee_args
    )
    if error is not None:
        return {**error, "tool": normalized_name, "arguments": call_arguments}
    assert config is not None

    payload, error = dispatch(config, normalized_name, call_arguments)
    if error is not None:
        return {**error, "tool": normalized_name, "arguments": call_arguments}
    assert payload is not None
    return payload
```
### 4.8 `integrations/gitee/tools/gitee_code_tool/__init__.py`

只读代码分析工具：仓库列表、目录浏览、文件读取、内容搜索、ref 对比。全部带 evidence mapper。

```python
"""Read-only Gitee code-analysis tools.

These tools wrap the Gitee MCP server's repository read surface:
``list_user_repos``, ``get_file_content``, ``search_files_by_content``, and
``compare_branches_tags``. They are all read-only and record evidence entries
so the final answer can cite the files and searches it used.
"""

from __future__ import annotations

from typing import Any

from core.domain.types.tools import ToolSurface
from core.tool import SideEffectLevel
from core.tool_framework import tool
from integrations.gitee.tools._shared import (
    AUTH_PROPERTIES,
    available,
    config_from_args,
    decode,
    dispatch,
    extract_params,
    file_view,
    record,
)


def _map_list_gitee_repos(
    evidence: dict[str, Any], output: dict[str, Any], _input: dict[str, Any]
) -> None:
    if not output.get("available"):
        return
    record(
        evidence,
        base="list_gitee_repos",
        label="Gitee Repositories",
        summary=f"{output.get('count', 0)} repositories",
    )


def _map_list_gitee_tree(
    evidence: dict[str, Any], output: dict[str, Any], _input: dict[str, Any]
) -> None:
    if not output.get("available") or output.get("kind") != "directory":
        return
    record(
        evidence,
        base="list_gitee_tree",
        label="Gitee Directory",
        summary=f"{output.get('count', 0)} items: {output.get('path') or '/'}",
    )


def _map_get_gitee_file(
    evidence: dict[str, Any], output: dict[str, Any], _input: dict[str, Any]
) -> None:
    if not output.get("available"):
        return
    if output.get("kind") == "file" and output.get("content"):
        record(
            evidence,
            base="get_gitee_file",
            label="Gitee File Contents",
            summary=f"File: {output.get('path')}",
        )


def _map_search_gitee_files(
    evidence: dict[str, Any], output: dict[str, Any], _input: dict[str, Any]
) -> None:
    if not output.get("available"):
        return
    record(
        evidence,
        base="search_gitee_files",
        label="Gitee Code Search",
        summary=f"{output.get('count', 0)} matches: {output.get('query', '')}",
    )


def _map_compare_gitee_refs(
    evidence: dict[str, Any], output: dict[str, Any], _input: dict[str, Any]
) -> None:
    if not output.get("available"):
        return
    record(
        evidence,
        base="compare_gitee_refs",
        label="Gitee Ref Comparison",
        summary=f"{output.get('base')}...{output.get('head')}",
    )


@tool(
    name="list_gitee_repos",
    source="gitee_mcp",
    side_effect_level=SideEffectLevel.READ_ONLY,
    concurrency_safe=True,
    description=(
        "List Gitee repositories the configured token can read. Use it to resolve "
        "the exact owner/repo before browsing code."
    ),
    use_cases=[
        "Finding the repository named in an alert or a user question",
        "Checking which private or organization repositories the token can read",
    ],
    surfaces=(ToolSurface.CHAT,),
    input_schema={
        "type": "object",
        "properties": {
            "visibility": {
                "type": "string",
                "enum": ["all", "public", "private"],
                "default": "all",
            },
            "affiliation": {
                "type": "string",
                "description": "owner, collaborator, organization_member, enterprise_member, or admin",
            },
            "q": {"type": "string", "description": "Optional repository search keywords."},
            "per_page": {"type": "integer", "minimum": 1, "maximum": 100, "default": 20},
            **AUTH_PROPERTIES,
        },
        "required": [],
    },
    is_available=available,
    extract_params=extract_params,
    evidence_mapper=_map_list_gitee_repos,
)
def list_gitee_repos(
    visibility: str = "all",
    affiliation: str = "",
    q: str = "",
    per_page: int = 20,
    gitee_url: str | None = None,
    gitee_mode: str | None = None,
    gitee_token: str | None = None,
    gitee_command: str | None = None,
    gitee_args: list[str] | None = None,
    **_kwargs: object,
) -> dict[str, object]:
    """List repositories visible to the configured Gitee token."""
    config, error = config_from_args(
        gitee_url, gitee_mode, gitee_token, gitee_command, gitee_args
    )
    if error is not None:
        return error
    assert config is not None

    arguments: dict[str, object] = {
        "visibility": visibility,
        "per_page": max(1, min(per_page, 100)),
    }
    if affiliation:
        arguments["affiliation"] = affiliation
    if q:
        arguments["q"] = q

    payload, error = dispatch(config, "list_user_repos", arguments)
    if error is not None:
        return error
    assert payload is not None

    decoded = decode(payload)
    repositories = decoded if isinstance(decoded, list) else []
    return {
        "source": "gitee_mcp",
        "available": True,
        "count": len(repositories),
        "repositories": repositories,
        "text": payload.get("text", ""),
    }


@tool(
    name="list_gitee_tree",
    source="gitee_mcp",
    side_effect_level=SideEffectLevel.READ_ONLY,
    concurrency_safe=True,
    description=(
        "List one directory level in a Gitee repository. Pass an empty path for "
        "the repository root, then descend into the directories the listing "
        "returns. Uses the Gitee contents API, which also serves single files."
    ),
    use_cases=[
        "Understanding repository structure before reading code",
        "Finding the directory that matches a stack frame or an alert",
        "Checking whether a config or workflow path exists on a branch",
    ],
    requires=["owner", "repo"],
    surfaces=(ToolSurface.CHAT,),
    input_schema={
        "type": "object",
        "properties": {
            "owner": {"type": "string"},
            "repo": {"type": "string"},
            "path": {
                "type": "string",
                "default": "",
                "description": "Directory path; empty means the repository root.",
            },
            "ref": {"type": "string", "description": "Branch, tag, or commit; defaults to the default branch."},
            **AUTH_PROPERTIES,
        },
        "required": ["owner", "repo"],
    },
    is_available=available,
    extract_params=extract_params,
    evidence_mapper=_map_list_gitee_tree,
)
def list_gitee_tree(
    owner: str,
    repo: str,
    path: str = "",
    ref: str = "",
    gitee_url: str | None = None,
    gitee_mode: str | None = None,
    gitee_token: str | None = None,
    gitee_command: str | None = None,
    gitee_args: list[str] | None = None,
    **_kwargs: object,
) -> dict[str, object]:
    """List one directory level in a Gitee repository."""
    config, error = config_from_args(
        gitee_url, gitee_mode, gitee_token, gitee_command, gitee_args
    )
    if error is not None:
        return error
    assert config is not None

    arguments: dict[str, object] = {"owner": owner, "repo": repo, "path": path}
    if ref:
        arguments["ref"] = ref

    payload, error = dispatch(config, "get_file_content", arguments)
    if error is not None:
        return error
    assert payload is not None

    decoded = decode(payload)
    if isinstance(decoded, list):
        return {
            "source": "gitee_mcp",
            "available": True,
            "kind": "directory",
            "owner": owner,
            "repo": repo,
            "path": path,
            "ref": ref,
            "count": len(decoded),
            "entries": decoded,
            "text": payload.get("text", ""),
        }

    kind = str(decoded.get("type") or "file") if isinstance(decoded, dict) else "file"
    return {
        "source": "gitee_mcp",
        "available": True,
        "kind": kind,
        "owner": owner,
        "repo": repo,
        "path": path,
        "ref": ref,
        "count": 0,
        "entries": [],
        "note": "The path is a file, not a directory. Use get_gitee_file to read it.",
        "text": payload.get("text", ""),
    }


@tool(
    name="get_gitee_file",
    source="gitee_mcp",
    side_effect_level=SideEffectLevel.READ_ONLY,
    concurrency_safe=True,
    description=(
        "Read a file from a Gitee repository at a branch, tag, or commit. Large "
        "files are truncated to max_chars; narrow with search_gitee_files first "
        "when the file is huge."
    ),
    use_cases=[
        "Reading application code referenced by an alert or stack trace",
        "Inspecting CI config, manifests, and deployment files",
        "Checking how a path looked on a branch or commit",
    ],
    requires=["owner", "repo", "path"],
    surfaces=(ToolSurface.CHAT,),
    input_schema={
        "type": "object",
        "properties": {
            "owner": {"type": "string"},
            "repo": {"type": "string"},
            "path": {"type": "string"},
            "ref": {"type": "string"},
            "max_chars": {
                "type": "integer",
                "minimum": 1000,
                "maximum": 200000,
                "default": 60000,
                "description": "Truncate the returned file content to this many characters.",
            },
            **AUTH_PROPERTIES,
        },
        "required": ["owner", "repo", "path"],
    },
    is_available=available,
    extract_params=extract_params,
    evidence_mapper=_map_get_gitee_file,
)
def get_gitee_file(
    owner: str,
    repo: str,
    path: str,
    ref: str = "",
    max_chars: int = 60000,
    gitee_url: str | None = None,
    gitee_mode: str | None = None,
    gitee_token: str | None = None,
    gitee_command: str | None = None,
    gitee_args: list[str] | None = None,
    **_kwargs: object,
) -> dict[str, object]:
    """Read a file from a Gitee repository."""
    config, error = config_from_args(
        gitee_url, gitee_mode, gitee_token, gitee_command, gitee_args
    )
    if error is not None:
        return error
    assert config is not None

    arguments: dict[str, object] = {"owner": owner, "repo": repo, "path": path}
    if ref:
        arguments["ref"] = ref

    payload, error = dispatch(config, "get_file_content", arguments)
    if error is not None:
        return error
    assert payload is not None

    view = file_view(decode(payload), max_chars=max(1000, min(max_chars, 200000)))
    return {
        "source": "gitee_mcp",
        "available": True,
        "owner": owner,
        "repo": repo,
        "path": path,
        "ref": ref,
        **view,
        "text": payload.get("text", ""),
    }


@tool(
    name="search_gitee_files",
    source="gitee_mcp",
    side_effect_level=SideEffectLevel.READ_ONLY,
    concurrency_safe=True,
    description=(
        "Search file contents in a Gitee repository. Use it to find the code "
        "behind an exception, a symbol, or a config key before reading files."
    ),
    use_cases=[
        "Finding source code related to failures, exceptions, and stack frames",
        "Tracing config, workflow, or application code that may explain an incident",
        "Locating call sites for a symbol before reading a specific file",
    ],
    requires=["owner", "repo", "query"],
    surfaces=(ToolSurface.CHAT,),
    input_schema={
        "type": "object",
        "properties": {
            "owner": {"type": "string"},
            "repo": {"type": "string"},
            "query": {"type": "string"},
            "ref": {"type": "string"},
            "limit": {"type": "integer", "minimum": 1, "maximum": 100, "default": 20},
            "paths": {
                "type": "string",
                "description": "Comma-separated paths or glob patterns to narrow the search.",
            },
            "before_context": {"type": "integer", "minimum": 0, "default": 0},
            "after_context": {"type": "integer", "minimum": 0, "default": 0},
            **AUTH_PROPERTIES,
        },
        "required": ["owner", "repo", "query"],
    },
    is_available=available,
    extract_params=extract_params,
    evidence_mapper=_map_search_gitee_files,
)
def search_gitee_files(
    owner: str,
    repo: str,
    query: str,
    ref: str = "",
    limit: int = 20,
    paths: str = "",
    before_context: int = 0,
    after_context: int = 0,
    gitee_url: str | None = None,
    gitee_mode: str | None = None,
    gitee_token: str | None = None,
    gitee_command: str | None = None,
    gitee_args: list[str] | None = None,
    **_kwargs: object,
) -> dict[str, object]:
    """Search file contents in a Gitee repository."""
    config, error = config_from_args(
        gitee_url, gitee_mode, gitee_token, gitee_command, gitee_args
    )
    if error is not None:
        return error
    assert config is not None

    arguments: dict[str, object] = {
        "owner": owner,
        "repo": repo,
        "query": query,
        "limit": max(1, min(limit, 100)),
    }
    if ref:
        arguments["ref"] = ref
    if paths:
        arguments["paths"] = paths
    if before_context:
        arguments["before_context"] = before_context
    if after_context:
        arguments["after_context"] = after_context

    payload, error = dispatch(config, "search_files_by_content", arguments)
    if error is not None:
        return error
    assert payload is not None

    decoded = decode(payload)
    matches = decoded if isinstance(decoded, list) else []
    return {
        "source": "gitee_mcp",
        "available": True,
        "owner": owner,
        "repo": repo,
        "query": query,
        "ref": ref,
        "count": len(matches),
        "matches": matches,
        "text": payload.get("text", ""),
    }


@tool(
    name="compare_gitee_refs",
    source="gitee_mcp",
    side_effect_level=SideEffectLevel.READ_ONLY,
    concurrency_safe=True,
    description=(
        "Compare two branches, tags, or commits in a Gitee repository. Use it to "
        "see which files changed between a known-good ref and the failing one."
    ),
    use_cases=[
        "Linking a deploy or release to the code that changed",
        "Checking what changed between two tags before an incident",
        "Narrowing a regression to a commit range",
    ],
    requires=["owner", "repo", "base", "head"],
    surfaces=(ToolSurface.CHAT,),
    input_schema={
        "type": "object",
        "properties": {
            "owner": {"type": "string"},
            "repo": {"type": "string"},
            "base": {"type": "string", "description": "Base branch, tag, or commit SHA."},
            "head": {"type": "string", "description": "Head branch, tag, or commit SHA."},
            **AUTH_PROPERTIES,
        },
        "required": ["owner", "repo", "base", "head"],
    },
    is_available=available,
    extract_params=extract_params,
    evidence_mapper=_map_compare_gitee_refs,
)
def compare_gitee_refs(
    owner: str,
    repo: str,
    base: str,
    head: str,
    gitee_url: str | None = None,
    gitee_mode: str | None = None,
    gitee_token: str | None = None,
    gitee_command: str | None = None,
    gitee_args: list[str] | None = None,
    **_kwargs: object,
) -> dict[str, object]:
    """Compare two Gitee refs."""
    config, error = config_from_args(
        gitee_url, gitee_mode, gitee_token, gitee_command, gitee_args
    )
    if error is not None:
        return error
    assert config is not None

    payload, error = dispatch(
        config,
        "compare_branches_tags",
        {"owner": owner, "repo": repo, "base": base, "head": head},
    )
    if error is not None:
        return error
    assert payload is not None

    return {
        "source": "gitee_mcp",
        "available": True,
        "owner": owner,
        "repo": repo,
        "base": base,
        "head": head,
        "compare": decode(payload),
        "text": payload.get("text", ""),
    }
```
---

## 5. 已有文件接线

### 5.1 `tools/registry_discovery.py`

把新 vendor 的工具包加入发现列表（放在 `integrations.github.tools` 之前）：

```python
    "integrations.gitee.tools",
    "integrations.github.tools",
```

### 5.2 `integrations/registry.py`

在 `sentry_mcp` 的 spec 附近加入：

```python
    IntegrationSpec(
        service="gitee_mcp",
        aliases=("gitee", "gitee mcp", "gitee-mcp"),
        has_verifier=True,
        direct_effective=True,
        setup_order=56,
        verify_order=60,
    ),
```

`setup_order` / `verify_order` 只影响展示与执行顺序；若当前分支已占用这些值，换一对未使用的即可。

### 5.3 `integrations/cli.py`

新增 setup handler：

```python
def _setup_gitee_mcp() -> None:
    from integrations.gitee.setup import GITEE_MCP_SETUP

    _run_spec_setup(GITEE_MCP_SETUP)
```

在 `_HANDLERS` 里加：

```python
    "gitee_mcp": _setup_gitee_mcp,
```

### 5.4 `integrations/catalog.py`

在 `sentry_mcp` 的 `add(...)` 后加入：

```python
    add(
        "gitee_mcp",
        _any_env(
            "GITEE_MCP_COMMAND",
            "GITEE_MCP_URL",
            "GITEE_MCP_AUTH_TOKEN",
            "GITEE_ACCESS_TOKEN",
        ),
    )
```

### 5.5 `integrations/_catalog_impl.py`

导入区加入：

```python
from config.constants.gitee import (
    GITEE_ACCESS_TOKEN_ENV,
    GITEE_MCP_ARGS_ENV,
    GITEE_MCP_AUTH_TOKEN_ENV,
    GITEE_MCP_COMMAND_ENV,
    GITEE_MCP_ENABLED_TOOLS_ENV,
    GITEE_MCP_MODE_ENV,
    GITEE_MCP_TIMEOUT_SECONDS_ENV,
    GITEE_MCP_URL_ENV,
)
from integrations.gitee import (
    DEFAULT_GITEE_MCP_URL,
    DEFAULT_GITEE_READ_TOOLS,
    build_gitee_mcp_config,
)
from integrations.gitee import classify as _classify_gitee_mcp
```

分类器映射加入：

```python
    "gitee_mcp": _classify_gitee_mcp,
```

在 `sentry_mcp` 的 env loader 块之后加入：

```python
    gitee_mode = os.getenv(GITEE_MCP_MODE_ENV, "streamable-http").strip().lower()
    gitee_mode = gitee_mode or "streamable-http"
    gitee_command = os.getenv(GITEE_MCP_COMMAND_ENV, "").strip()
    gitee_token = (
        resolve_env_credential(GITEE_MCP_AUTH_TOKEN_ENV)
        or resolve_env_credential(GITEE_ACCESS_TOKEN_ENV)
    )
    gitee_url = os.getenv(GITEE_MCP_URL_ENV, "").strip()
    if gitee_mode != "stdio" and gitee_token and not gitee_url:
        gitee_url = DEFAULT_GITEE_MCP_URL
    if (gitee_mode == "stdio" and gitee_command) or (
        gitee_mode != "stdio" and gitee_url and gitee_token
    ):
        try:
            gitee_timeout_raw = os.getenv(GITEE_MCP_TIMEOUT_SECONDS_ENV, "").strip()
            try:
                gitee_timeout = (
                    max(1.0, float(gitee_timeout_raw)) if gitee_timeout_raw else 30.0
                )
            except ValueError:
                gitee_timeout = 30.0
            gitee_config = build_gitee_mcp_config(
                {
                    "url": gitee_url,
                    "mode": gitee_mode,
                    "command": gitee_command,
                    "args": [
                        part
                        for part in os.getenv(GITEE_MCP_ARGS_ENV, "").strip().split()
                        if part
                    ],
                    "auth_token": gitee_token,
                    "timeout_seconds": gitee_timeout,
                    "enabled_tools": os.getenv(GITEE_MCP_ENABLED_TOOLS_ENV, "").strip()
                    or DEFAULT_GITEE_READ_TOOLS,
                }
            )
            integrations.append(
                _active_env_record(
                    "gitee_mcp",
                    {
                        **gitee_config.model_dump(exclude={"integration_id"}),
                        "connection_verified": True,
                    },
                )
            )
        except Exception as exc:
            _report_env_loader_failure(exc, integration="gitee_mcp")
```

### 5.6 `integrations/effective_models.py`

```python
    gitee_mcp: EffectiveIntegrationEntry | None = None
```

### 5.7 `config/constants` 的包级再导出（可选）

集成内部直接 `from config.constants.gitee import ...`，和 `sentry_mcp` 的做法一致，所以这一步不影响运行。若要让 `from config.constants import GITEE_MCP_URL_ENV` 也可用，在 `config/constants/exports.py` 加入：

```python
    "GITEE_ACCESS_TOKEN_ENV": "gitee",
    "GITEE_MCP_ARGS_ENV": "gitee",
    "GITEE_MCP_AUTH_TOKEN_ENV": "gitee",
    "GITEE_MCP_COMMAND_ENV": "gitee",
    "GITEE_MCP_ENABLED_TOOLS_ENV": "gitee",
    "GITEE_MCP_MODE_ENV": "gitee",
    "GITEE_MCP_TIMEOUT_SECONDS_ENV": "gitee",
    "GITEE_MCP_URL_ENV": "gitee",
```

并在 `config/constants/__init__.py` 的 `TYPE_CHECKING` 区加对应的 `from config.constants.gitee import (...) as ...`。

### 5.8 `.env.example`

```dotenv
# Gitee MCP (read-only repository code, files, search, and ref comparison)
GITEE_MCP_MODE=streamable-http
GITEE_MCP_URL=https://api.gitee.com/mcp
GITEE_MCP_AUTH_TOKEN=
GITEE_MCP_COMMAND=
GITEE_MCP_ARGS=
GITEE_MCP_TIMEOUT_SECONDS=30
GITEE_MCP_ENABLED_TOOLS=list_user_repos,get_file_content,search_files_by_content,compare_branches_tags,get_diff_files,list_repo_pulls,list_repo_issues,get_repo_issue_detail,list_comments
# GITEE_ACCESS_TOKEN=   # stdio fallback only; same token as GITEE_MCP_AUTH_TOKEN
```

---

## 6. 环境变量与配置

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `GITEE_MCP_MODE` | `streamable-http` | `streamable-http` / `sse` / `stdio` |
| `GITEE_MCP_URL` | `https://api.gitee.com/mcp` | 远程 MCP 端点 |
| `GITEE_MCP_AUTH_TOKEN` | 空 | Gitee 个人访问令牌，远程模式必填；写入 keyring |
| `GITEE_ACCESS_TOKEN` | 空 | stdio 模式使用的同名令牌；也作为 `GITEE_MCP_AUTH_TOKEN` 的回退 |
| `GITEE_MCP_COMMAND` | 空 | stdio 可执行文件，例如 `npx` 或 `mcp-gitee.exe` |
| `GITEE_MCP_ARGS` | 空 | stdio 参数，空格分隔，例如 `-y @gitee/mcp-gitee@latest` |
| `GITEE_MCP_TIMEOUT_SECONDS` | `30` | 单次 MCP 操作超时 |
| `GITEE_MCP_ENABLED_TOOLS` | 只读白名单 | 逗号分隔；远程通过 `X-MCP-Enabled-Tools` 发送，stdio 通过 `ENABLED_TOOLSETS` 发送 |

默认只读白名单：

```text
list_user_repos
get_file_content
search_files_by_content
compare_branches_tags
get_diff_files
list_repo_pulls
list_repo_issues
get_repo_issue_detail
list_comments
```

要启用写工具，必须显式把这些工具名加入 `GITEE_MCP_ENABLED_TOOLS`；`call_gitee_tool` 每次都会要求审批。

---

## 7. 使用流程

### 7.1 安装与验证

远程模式（推荐）：

```bash
opensre integrations setup gitee
# URL 保持默认 https://api.gitee.com/mcp
# Token 填 Gitee 个人访问令牌（仓库读权限即可）
opensre integrations verify gitee
```

纯环境变量方式：

```dotenv
GITEE_MCP_MODE=streamable-http
GITEE_MCP_URL=https://api.gitee.com/mcp
GITEE_MCP_AUTH_TOKEN=your_gitee_personal_access_token
```

stdio 兜底：

```dotenv
GITEE_MCP_MODE=stdio
GITEE_MCP_COMMAND=npx
GITEE_MCP_ARGS=-y @gitee/mcp-gitee@latest
GITEE_MCP_AUTH_TOKEN=your_gitee_personal_access_token
# 或 GITEE_ACCESS_TOKEN=...
```

### 7.2 在交互式 Shell 中提问

```text
在 Gitee 的 owner/repo 里分析一下 OrderService 的超时处理，
先看目录结构，再搜索 TimeoutError，最后读相关文件。
```

预期工具调用：

```text
list_gitee_repos / list_gitee_tree
  -> search_gitee_files(owner, repo, query="TimeoutError")
  -> get_gitee_file(owner, repo, path="src/order/service.py", ref="master")
  -> compare_gitee_refs(owner, repo, base="v1.2.0", head="v1.3.0")
```

### 7.3 典型调查路径

1. 告警或提问里没有明确 `owner/repo`：先 `list_gitee_repos`；
2. 有 `owner/repo`：`list_gitee_tree` 看根目录；
3. 根据异常类名、栈帧、配置键：`search_gitee_files`；
4. 搜索结果给出文件路径后：`get_gitee_file` 读取；
5. 怀疑版本回归：`compare_gitee_refs` 比较 tag / commit；
6. 需要 Issue/PR 上下文：`list_gitee_tools` 发现工具后用 `call_gitee_tool` 调用（会要求审批）。

---

## 8. 测试

已新增：

- `tests/integrations/test_gitee_mcp.py`：配置模型、环境变量、stdio/远程模式、token 回退、只读白名单、校验、classify 与别名；
- `tests/tools/test_gitee_mcp_tool.py`：工具契约、可用性、参数注入、base64 文件、目录、搜索、ref 对比、通用桥、错误路径和 `_shared` 解码。

覆盖重点：

| 测试 | 重点 |
| --- | --- |
| 配置 | 默认远程 URL、token 缺失、stdio 模式、`GITEE_ACCESS_TOKEN` 回退、超时解析 |
| 白名单 | 空 `GITEE_MCP_ENABLED_TOOLS` 仍回退只读白名单 |
| 校验 | mock `list_gitee_mcp_tools`；缺少 `get_file_content` / `search_files_by_content` 时失败 |
| 共享层 | `file_view` 的 base64 解码、目录列表、`max_chars` 截断；`decode` 优先 `structured_content` |
| 工具层 | mock `dispatch`；验证目录/文件/搜索/对比输出形状和 evidence 条目 |
| 接线 | `resolve_management_service("gitee") == "gitee_mcp"`；`_HANDLERS["gitee_mcp"]` 存在；工具注册表发现 7 个工具 |
| 打包 | `INTEGRATION_TOOL_PACKAGES` 包含 `integrations.gitee.tools`；`runtime_hidden_imports` 覆盖新模块 |

运行：

```bash
uv run ruff check integrations/gitee tests/integrations/test_gitee_mcp.py tests/tools/test_gitee_mcp_tool.py
uv run mypy integrations/gitee config/constants/gitee.py
uv run pytest tests/integrations/test_gitee_mcp.py tests/tools/test_gitee_mcp_tool.py -q
```

---

## 9. 限制与后续演进

1. **没有完整 commit log 工具**。`compare_branches_tags` 和 `get_diff_files` 能看变更，但不能替代 `git log --oneline`。需要完整历史时，仍建议本地 clone，或后续在 `integrations/gitee/` 里增加 REST 的 `list_gitee_commits`。
2. **目录浏览依赖 `get_file_content` 对目录路径返回列表**。这是 Gitee contents API 的行为；若当前 MCP Server 版本只接受文件路径，`list_gitee_tree` 会返回“path is a file”，此时用 `search_gitee_files` 或本地 clone。
3. **远程 MCP 的可用性和额度以 Gitee 官方为准**。远程不可用时切 `GITEE_MCP_MODE=stdio`。
4. **默认只读**。要调用写工具，必须显式放宽 `GITEE_MCP_ENABLED_TOOLS`；通用桥每次调用都需要审批。
5. **大文件会截断**。`get_gitee_file` 默认 `max_chars=60000`，先搜索再读文件。
6. **Token 最小权限**。只给仓库读权限，定期轮换；不要把 token 写进仓库或 `.env.example`。
7. **证据粒度**。当前 evidence 记录文件路径、目录和搜索词，不记录文件内容片段；如果报告需要引用具体代码行，可在 `_map_get_gitee_file` 里追加 `snippet`（截断后的前 N 行）。

---

## 10. 参考

- Gitee MCP Server 帮助中心：<https://help.gitee.com/ai-productivity/mcp-server>
- Gitee MCP Server 源码：<https://gitee.com/oschina/mcp-gitee>
- Gitee 个人访问令牌：<https://gitee.com/profile/personal_access_tokens>
- OpenSRE 工具放置策略：[tool-placement-policy.md](tool-placement-policy.md)
- OpenSRE 集成开发检查清单：[adding-tools-and-integrations.md](adding-tools-and-integrations.md)
