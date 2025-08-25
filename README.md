## Home Page Example

```tsx
"use client"
import { useState } from "react"

export default function Home() {
  const [url, setUrl] = useState("")
  const [result, setResult] = useState<any>(null)
  const [loading, setLoading] = useState(false)

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    setLoading(true)
    setResult(null)

    const res = await fetch("/api/analyze", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ url }),
    })

    const data = await res.json()
    setResult(data)
    setLoading(false)
  }

  return (
    <main style={{ padding: "40px", fontFamily: "Arial" }}>
      <h1>🔎 Website SEO & Link Analyzer</h1>
      <form onSubmit={handleSubmit}>
        <input 
          type="text" 
          placeholder="Enter website URL (https://example.com)" 
          value={url}
          onChange={(e) => setUrl(e.target.value)}
          style={{ padding: "10px", width: "350px" }}
        />
        <button type="submit" style={{ padding: "10px 20px", marginLeft: "10px" }}>
          Scan
        </button>
      </form>

      {loading && <p>⏳ Scanning website...</p>}

      {result && (
        <div style={{ marginTop: "20px" }}>
          <h2>Results for {result.url}</h2>
          <h3>SEO Score: {result.score}/100</h3>

          <ul>
            <li><b>Title:</b> {result.title}</li>
            <li><b>Description:</b> {result.description}</li>
            <li><b>Google Verification:</b> {result.googleVerification}</li>
            <li><b>Total Links Found:</b> {result.totalLinks}</li>
            <li><b>Broken Links:</b> {result.brokenLinks.length > 0 ? result.brokenLinks.join(", ") : "None 🎉"}</li>
            <li><b>Word Count:</b> {result.wordCount}</li>
            <li><b>Headings (H1):</b> {result.headings.join(" | ") || "No H1 tags"}</li>
          </ul>
        </div>
      )}
    </main>
  )
}
## API Route Example

```ts
import { NextResponse } from "next/server"
import * as cheerio from "cheerio"

export async function POST(req: Request) {
  try {
    const { url } = await req.json()
    const response = await fetch(url)
    const html = await response.text()
    const $ = cheerio.load(html)

    const title = $("title").text() || "No title found"
    const description = $('meta[name="description"]').attr("content") || "No description found"
    const googleVerification = $('meta[name="google-site-verification"]').attr("content") || "Not found"
    const wordCount = $("body").text().split(/\s+/).length

    const links: string[] = []
    $("a").each((_, el) => {
      const href = $(el).attr("href")
      if (href && href.startsWith("http")) links.push(href)
    })

    const brokenLinks: string[] = []
    for (const link of links.slice(0, 10)) {
      try {
        const res = await fetch(link, { method: "HEAD" })
        if (!res.ok) brokenLinks.push(link)
      } catch {
        brokenLinks.push(link)
      }
    }

    const headings: string[] = []
    $("h1").each((_, el) => headings.push($(el).text()))

    let score = 100
    if (title === "No title found") score -= 20
    if (description === "No description found") score -= 20
    if (headings.length === 0) score -= 10
    if (brokenLinks.length > 0) score -= Math.min(brokenLinks.length * 5, 20)
    if (wordCount < 300) score -= 10
    if (googleVerification === "Not found") score -= 5
    if (score < 0) score = 0

    return NextResponse.json({
      url,
      title,
      description,
      googleVerification,
      totalLinks: links.length,
      brokenLinks,
      wordCount,
      headings,
      score,
    })
  } catch (error) {
    return NextResponse.json({ error: "Failed to analyze website" }, { status: 500 })
  }
}
