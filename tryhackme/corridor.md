Corridor – TryHackMe CTF Write-up

🧠 Room Premise

    "You find yourself in a strange corridor. Can you make your way back?"

This room focuses on identifying and exploiting IDOR (Insecure Direct Object References) vulnerabilities. Pay close attention to the hexadecimal values in URLs—they’re not just random. They’re a clue.
🕸️ Initial Observations

Right off the bat, this is clearly a web application challenge. Instead of running typical Nmap scans, I jump straight to web-based enumeration. The interface displays multiple clickable doors—each leads to a new page with a suspicious hash.

Upon further analysis, these hashes are MD5 representations of page numbers. By reversing them (e.g., using CrackStation or manual MD5 lookups), we identify predictable URL patterns vulnerable to IDOR.

Trying non-displayed values like 0, 14, or even /admin, uncovers areas of the application we shouldn’t normally access—confirming the vulnerability.
🚩 Key Takeaways

    This room serves as a reminder of how simple logic flaws, like IDOR, still plague real-world applications.

    According to OWASP, Broken Access Control is one of the most exploited vulnerabilities today—found in 94% of tested apps.

    Security by obscurity (like hiding page numbers behind hashes) is not real protection. Determined attackers will find ways in.

This was a short but effective refresher on one of the most common and dangerous web flaws. IDOR is simple to discover, yet devastating when exploited. Always validate user access on the server-side—never rely on hidden URLs or hashes alone.
