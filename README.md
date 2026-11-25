# Smart Solar Agri-Hub Swarm — Autonomous Agri-Climate Research Agent (Tavily Track)

## Overview
A decentralized multi-agent system built by **Climate Nanos Enterprise** integrating IoT telemetry, adaptive solar microgrid control logic, automated agricultural utility management (temperature-controlled cold chain preservation and automated irrigation), and a **Tavily-powered Agri-Climate Research Agent**.

During operation, the **Tavily Research Agent** equips the swarm with live web intelligence—retrieving regional weather advisories, pest/disease mitigation guidelines, post-harvest cold-storage parameters, and agricultural market trends—to guide automated hub decisions and operator advisories with verifiable citations.

## Preexisting Work & Attribution
* **Preexisting Work**: Baseline physical microgrid architecture, hardware component integration (ESP32/Arduino telemetry, sensors, and off-grid solar PV setup), and baseline hardware documentation built prior to the sprint.
* **Sprint Work**: Autonomous **Tavily Agri-Climate Research Agent** (`agents/tavily_agri_research_agent.py`) featuring multi-angle query decomposition, live Tavily web search & content extraction (`tavily-python`), offline demo fallback mode, cited research brief synthesis, and swarm integration documentation.

## Repository Structure
* `/agents`: Python multi-agent research modules, including `tavily_agri_research_agent.py` (Tavily live search & extraction agent).
* `/firmware`: Embedded code for microcontrollers (ESP32/Arduino).
* `/dashboard`: Web interface and telemetry dashboard scripts.
* `/docs`: System architecture diagrams and technical documentation.

## How the Tavily Research Agent Works
1. **Query Decomposition:** Breaks a broad hub or crop-management prompt into targeted sub-queries covering agronomic best practices, regional weather/pest alerts, and solar cold-chain optimization.
2. **Live Web Retrieval (`TavilyClient.search` & `TavilyClient.extract`):** Queries real-time web sources, deduplicates URLs, and extracts full-page agronomic/weather context when needed.
3. **Zero-Config Fallback Mode:** Automatically runs in structured demo mode if `TAVILY_API_KEY` is omitted, and switches seamlessly to live web intelligence whenever `TAVILY_API_KEY` is configured.
4. **Cited Decision Briefs:** Outputs structured findings with source URLs ready for swarm orchestrators or CLI inspection.

## Setup & Execution

### 1. Run the Tavily Agri-Climate Research Agent
```bash
# Install dependencies
pip install -r requirements.txt

# Optional: Set your free Tavily API key from [https://app.tavily.com](https://app.tavily.com) for live search
# (If omitted, the agent automatically runs in built-in demo fallback mode)
export TAVILY_API_KEY="tvly-your-api-key-here"

# Execute the research agent
python agents/tavily_agri_research_agent.py

---

### 2. Supporting Files to Commit Alongside `README.md`

**File: `agents/tavily_agri_research_agent.py`**
```python
import os
from typing import Any, Dict, List

try:
    from tavily import TavilyClient
except ImportError:
    TavilyClient = None


class TavilyAgriResearchAgent:
    """Agri-Climate Research Agent powered by Tavily with an offline demo fallback."""

    def __init__(self, api_key: str | None = None) -> None:
        self.api_key = (api_key or os.getenv("TAVILY_API_KEY") or "").strip()
        self.live_mode = bool(self.api_key and TavilyClient is not None)
        self.client = TavilyClient(api_key=self.api_key) if self.live_mode else None

    def decompose_query(self, topic: str, region: str = "Nigeria") -> List[str]:
        """Break a broad agritech/hub prompt into focused sub-queries."""
        clean_topic = topic.strip()
        return [
            f"{clean_topic} {region} best practices guidelines",
            f"{clean_topic} {region} weather pest market alerts",
            f"{clean_topic} cold storage humidity solar optimization",
        ]

    def _fallback_research(
        self, topic: str, region: str, sub_queries: List[str]
    ) -> Dict[str, Any]:
        """Structured fallback output when TAVILY_API_KEY is not configured."""
        return {
            "mode": "demo_fallback (set TAVILY_API_KEY for live web search)",
            "topic": topic,
            "region": region,
            "sub_queries": sub_queries,
            "findings": [
                {
                    "sub_query": sub_queries[0],
                    "title": "Post-Harvest Cold Chain Guidelines — FAO & Agritech Extension",
                    "url": "https://www.fao.org/post-harvest",
                    "snippet": (
                        "Maintaining consistent storage temperatures (10-13°C for ripe tomatoes "
                        "and 7-10°C for peppers at 90-95% RH) with sanitized airflow reduces "
                        "post-harvest spoilage and fungal decay."
                    ),
                },
                {
                    "sub_query": sub_queries[1],
                    "title": f"Regional Agrometeorological Advisory — {region}",
                    "url": "https://nimet.gov.ng",
                    "snippet": (
                        "High ambient humidity increases early/late blight risk in solanaceous "
                        "crops; automated ventilation and moisture monitoring are recommended."
                    ),
                },
                {
                    "sub_query": sub_queries[2],
                    "title": "Off-Grid Solar Agri-Hub Load Scheduling",
                    "url": "https://climate-nanos-enterprise.netlify.app",
                    "snippet": (
                        "Prioritize peak solar generation windows (10:00-15:00) for precooling "
                        "thermal mass and running automated irrigation cycles."
                    ),
                },
            ],
            "extracted_pages": [],
        }

    def research(
        self,
        topic: str,
        region: str = "Nigeria",
        max_results_per_query: int = 3,
        extract_top_urls: bool = False,
    ) -> Dict[str, Any]:
        """Run multi-query Tavily search (or fallback if no API key is present)."""
        sub_queries = self.decompose_query(topic=topic, region=region)

        if not self.live_mode or self.client is None:
            return self._fallback_research(topic, region, sub_queries)

        findings: List[Dict[str, str]] = []
        seen_urls: set[str] = set()

        for sub_q in sub_queries:
            res = self.client.search(
                query=sub_q,
                search_depth="basic",
                max_results=max_results_per_query,
                include_answer="basic",
            )
            if res.get("answer"):
                findings.append(
                    {
                        "sub_query": sub_q,
                        "title": "Tavily Quick Synthesis",
                        "url": "",
                        "snippet": res["answer"],
                    }
                )

            for item in res.get("results", []):
                url = item.get("url", "")
                if url and url in seen_urls:
                    continue
                if url:
                    seen_urls.add(url)
                findings.append(
                    {
                        "sub_query": sub_q,
                        "title": item.get("title", "Untitled Source"),
                        "url": url,
                        "snippet": (item.get("content") or "").strip(),
                    }
                )

        extracted_pages: List[Dict[str, str]] = []
        if extract_top_urls:
            urls_to_extract = list(seen_urls)[:2]
            if urls_to_extract:
                ext = self.client.extract(urls=urls_to_extract)
                for page in ext.get("results", []):
                    raw = (page.get("raw_content") or "").strip()
                    extracted_pages.append(
                        {
                            "url": page.get("url", ""),
                            "excerpt": raw[:1200],
                        }
                    )

        return {
            "mode": "live_tavily",
            "topic": topic,
            "region": region,
            "sub_queries": sub_queries,
            "findings": findings,
            "extracted_pages": extracted_pages,
        }

    def format_brief(self, report: Dict[str, Any]) -> str:
        """Format research output for swarm orchestrators or CLI inspection."""
        lines = [
            f"=== TAVILY AGRI-HUB RESEARCH BRIEF [{report['mode']}] ===",
            f"Topic: {report['topic']} | Region: {report['region']}",
            "",
            "Sub-queries executed:",
        ]
        for q in report["sub_queries"]:
            lines.append(f"  - {q}")

        lines.append("\nKey Findings & Sources:")
        for idx, item in enumerate(report["findings"], start=1):
            lines.append(f"{idx}. [{item['title']}]")
            if item["snippet"]:
                lines.append(f"   {item['snippet'][:400]}")
            if item["url"]:
                lines.append(f"   Source: {item['url']}")

        return "\n".join(lines)


if __name__ == "__main__":
    agent = TavilyAgriResearchAgent()
    sample_report = agent.research(
        topic="tomato and pepper post-harvest cold storage and blight prevention",
        region="Ogun State, Nigeria",
        max_results_per_query=2,
    )
    print(agent.format_brief(sample_report))
