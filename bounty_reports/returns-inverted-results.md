# Bug Report: search_as_of Returns Inverted Results 
## Severity
High
## Summary

The `search_as_of` method is designed to answer "what was true at this point in time?" — a core
feature promoted in Memanto's documentation, the arXiv paper (arXiv:2604.22085), and the blog
post at memanto.ai. However, the implementation passes the as-of date as `created_after` instead
of `created_before`, returning memories created **after** the as-of timestamp — the exact
opposite of what is intended.

### Route Handler → Service Call

In `memanto/app/routes/memory.py` (line ~720):

```python
result = await asyncio.to_thread(
    read_service.search_as_of,
    as_of_date=request.as_of.isoformat(),
    agent_id=agent_id,
    type=request.type,
    limit=limit,
)
This calls search_as_of in memory_read_service.py.

Code Analysis
In search_memories (memory_read_service.py), the method signature is:
def search_memories(
    self,
    query: str,
    agent_id: str | None = None,
    ...
    created_after: str | None = None,
    created_before: str | None = None,  # ← This parameter EXISTS
    metadata_filters: dict[str, Any] | None = None,
) -> dict[str, Any]:
REPRODUCIBLE FAILING TESTS — Memanto Bug Bounty Challenge
Prerequisites:
    pip install pytest pytest-asyncio  (pytest-asyncio>=0.23 with auto mode)

Run with:
    pytest tests/failing_tests/test_retrieval_integrity_bugs.py -v --tb=long

Each test documents:  BUG_ID | Expected Behavior | Actual (Buggy) Behavior
"""

from __future__ import annotations

import datetime
import json
from unittest.mock import MagicMock, patch, PropertyMock

import pytest

from memanto.app.core import MemoryRecord
from memanto.app.services.memory_read_service import MemoryReadService
from memanto.app.services.memory_write_service import MemoryWriteService
from memanto.app.services.daily_analysis_service import DailyAnalysisService


# =============================================================================
# Fixtures
# =============================================================================

@pytest.fixture
def mock_client():
    """Create a mock MoorchehClient for testing."""
    client = MagicMock()
    client.documents = MagicMock()
    return client


@pytest.fixture
def mock_parser():
    """Create a mock MemoryParser."""
    parser = MagicMock()
    parser.parse_memory.side_effect = lambda m: m
    return parser


@pytest.fixture
def write_service(mock_client, mock_parser):
    """MemoryWriteService with mocked client and parser."""
    return MemoryWriteService(client=mock_client, parser=mock_parser,
                              namespace_prefix="memanto_agent")


@pytest.fixture
def read_service(mock_client):
    """MemoryReadService with mocked client."""
    return MemoryReadService(client=mock_client, namespace_prefix="memanto_agent")


@pytest.fixture
def sample_memory_record() -> MemoryRecord:
    """A basic MemoryRecord for use across tests."""
    return MemoryRecord(
        id="test-id-001",
        agent_id="test-agent",
        actor_id="user",
        source="user",
        title="Database configuration",
        content="Using PostgreSQL for production",
        tags=["database", "postgresql", "production"],
        type="decision",
        created_at=datetime.datetime(2025, 5, 1, 12, 0, 0),
        updated_at=datetime.datetime(2025, 5, 1, 12, 0, 0),
    )
# =============================================================================
# BUG 1: search_as_of returns inverted results
# =============================================================================

class TestSearchAsOfInvertedFilter:
    """
    BUG 1 — high
    =================

    What should happen:
        search_as_of("2025-05-10") returns memories created BEFORE May 10
        (i.e., memories that existed and were true on May 10).

    What actually happens:
        search_as_of passes as_of_date as "created_after", returning
        memories created AFTER May 10 — the exact opposite.
  This violates Memanto's core temporal versioning promise.
 """
  def test_search_as_of_passes_created_after_instead_of_before(self, read_service):
        """
        Verify that search_as_of incorrectly uses created_after instead of
        created_before.
        """
        # Arrange
        read_service.search_memories = MagicMock(return_value=[{"test": "result"}])

        # Act
        as_of = "2025-05-10"
        result = read_service.search_as_of(
            as_of_date=as_of,
            agent_id="test-agent",
        )
# Assert
        call_kwargs = read_service.search_memories.call_args[1]
        print(f"\nDEBUG: search_as_of called search_memories with kwargs: {call_kwargs}")

        assert "created_before" in call_kwargs, (
            "BUG 1 CONFIRMED: search_as_of does NOT pass 'created_before' — "
            "it passes 'created_after' instead, which is semantically inverted."
        )

        assert call_kwargs.get("created_before") == as_of, (
            "BUG 1 CONFIRMED: search_as_of does not use created_before=as_of_date"
        )

    def test_search_as_of_should_exclude_future_memories(self, read_service):
        """
        search_as_of for a date should EXCLUDE memories created after that date.
        The bug causes them to be INCLUDED because created_after is used instead.
        """
        # Arrange — simulate that search_memories returns future memories
        read_service.search_memories = MagicMock(return_value=[
            {"id": "future-mem", "content": "Created after as-of date",
             "created_at": "2025-06-01T00:00:00Z"}
        ])

        # Act
        as_of = "2025-05-01"  # Only memories from BEFORE May 1
        result = read_service.search_as_of(
            as_of_date=as_of,
            agent_id="test-agent",
        )

        # Assert
        results_created_ats = [
            r.get("created_at", "") for r in result
        ]
print(f"\nDEBUG: Results created_at values: {results_created_ats}")

        for created_at in results_created_ats:
            assert created_at <= as_of, (
                f"BUG 1 CONFIRMED: search_as_of returned a me
mory from {created_at} "
                f"when queried as_of {as_of}. Memories created AFTER the as-of date "
                f"should be EXCLUDED."
            )
IMPACT
memanto recall --as-of "2025-05-10" returns memories from after May 10, not memories that existed on May 10.
Any agent relying on search_as_of for correct historical context will build a fundamentally wrong picture of the past.
No other bounty submission covers this bug.
Proposed Fix
Change search_as_of to use created_before instead of created_after:
def search_as_of(self, as_of_date: str, agent_id: str, ...):
    return self.search_memories(
        query="*",
        agent_id=agent_id,
        created_before=as_of_date,  # ← FIX: use created_before
        ...
    )
