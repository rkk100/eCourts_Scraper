import argparse
import json
import os
import time
from datetime import datetime, timedelta
from pathlib import Path

from bs4 import BeautifulSoup
from webdriver_manager.chrome import ChromeDriverManager
from selenium import webdriver
from selenium.common.exceptions import NoSuchElementException, TimeoutException
from selenium.webdriver.chrome.service import Service as ChromeService
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


# -------------------- Utilities --------------------

def ensure_output_dir(path: str = "output"):
    Path(path).mkdir(parents=True, exist_ok=True)
    return Path(path)


def now_date_str():
    return datetime.now().strftime("%Y-%m-%d")


def is_today_or_tomorrow(date_str: str):
    """Given a date string in ISO-like format or common formats, check if it's today or tomorrow."""
    try:
        dt = datetime.strptime(date_str.strip(), "%Y-%m-%d")
    except Exception:
        # try common other formats
        for fmt in ("%d-%m-%Y", "%d/%m/%Y", "%d %b %Y", "%d %B %Y"):
            try:
                dt = datetime.strptime(date_str.strip(), fmt)
                break
            except Exception:
                dt = None
        if dt is None:
            return False, None
    today = datetime.now().date()
    if dt.date() == today:
        return True, "today"
    if dt.date() == (today + timedelta(days=1)):
        return True, "tomorrow"
    return False, dt.date().isoformat()


# -------------------- Selenium helpers --------------------

def make_driver(headless: bool = False, window_size: str = "1200,900"):
    options = webdriver.ChromeOptions()
    if headless:
        options.add_argument("--headless=new")
        options.add_argument("--disable-gpu")
    options.add_argument(f"--window-size={window_size}")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")
    service = ChromeService(ChromeDriverManager().install())
    driver = webdriver.Chrome(service=service, options=options)
    return driver


# -------------------- Core flows --------------------

def open_cnr_search_and_get(driver, cnr: str, wait_seconds: int = 120):
    """
    Opens the eCourts CNR search page, fills in the CNR and waits for user to solve captcha then submit.
    After submit, waits for the result and returns the page HTML.
    """
    url = "https://services.ecourts.gov.in/ecourtindia_v6/?p=search%2Fsearch&subtype=cnr"
    driver.get(url)
    print("Opened eCourts CNR search page in browser. The page may show a CAPTCHA.\n")

    try:
        # try to find the CNR input - the site updates and ID/names vary; try common ones
        possible_selectors = [
            (By.ID, "cnrNo"),
            (By.NAME, "cnrNo"),
            (By.CSS_SELECTOR, "input[name='cnr_no']"),
            (By.CSS_SELECTOR, "input[type='text']"),
        ]
        input_element = None
        for by, sel in possible_selectors:
            try:
                input_element = WebDriverWait(driver, 5).until(
                    EC.presence_of_element_located((by, sel))
                )
                break
            except Exception:
                continue
        if input_element is None:
            print("Could not find the CNR input automatically. Please paste the CNR into the browser manually, then solve CAPTCHA and submit the form.")
        else:
            input_element.clear()
            input_element.send_keys(cnr)
            print(f"Filled CNR value: {cnr}")

        print("Please solve the CAPTCHA in the opened browser window and then click the Search/Submit button on the page.\nAfter submitting, come back here and press Enter to continue.")
        input("Press Enter here when you have submitted the form in the browser...")

        # Wait for results to appear - the result panel may have selectors like '.result' or '#caseDetail'
        try:
            WebDriverWait(driver, wait_seconds).until(
                EC.presence_of_element_located((By.CSS_SELECTOR, "div#caseDetails, div.case-details, .result, .case-detail"))
            )
        except TimeoutException:
            print("Timed out waiting for a results panel; attempting to grab page content anyway.")

        html = driver.page_source
        return html

    except Exception as e:
        print("Error while interacting with CNR search page:", e)
        return driver.page_source


def parse_case_detail_html(html: str):
    soup = BeautifulSoup(html, "lxml")
    data = {}

    # Try to extract party names, court name, next hearing etc. Structure varies across districts.
    try:
        # Court name
        court = None
        for sel in ("h3.court-name", "#courtName", ".court-header", "div.court-name"):
            el = soup.select_one(sel)
            if el and el.get_text(strip=True):
                court = el.get_text(strip=True)
                break
        if not court:
            # fallback: find any strong text that looks like court
            text = soup.get_text(separator="|", strip=True)
            if "Court" in text:
                court = text.split("|")[0]
        data["court_name"] = court

        # Next hearing / listed date - try common labels
        next_hearing = None
        for label in ("Next Hearing Date", "Next Hearing", "Next Date", "Next Date of Hearing", "Hearing Date"):
            candidate = soup.find(text=lambda t: t and label.lower() in t.lower())
            if candidate:
                parent = candidate.parent
                # look for nearby date
                sibling = parent.find_next(string=True)
                if sibling and any(ch.isdigit() for ch in sibling):
                    next_hearing = sibling.strip()
                    break
        # alternative: search for patterns like 14-10-2025
        if not next_hearing:
            import re
            m = re.search(r"(\d{1,2}[-/ ]\d{1,2}[-/ ]\d{4})", soup.get_text())
            if m:
                next_hearing = m.group(1)
        data["next_hearing_raw"] = next_hearing

        # Serial number might be in cause list entries; look for 'Serial No' or 'Sl. No.' near numbers
        serial = None
        for key in ("Serial No", "Sl. No", "S. No", "Sl No", "Serial"):
            el = soup.find(text=lambda t: t and key.lower() in t.lower())
            if el:
                # attempt to find number nearby
                nearby = el.parent.find_next(string=True)
                if nearby and any(ch.isdigit() for ch in nearby):
                    serial = nearby.strip()
                    break
        data["serial_no"] = serial

        # Search for a PDF link in the page
        pdf_link = None
        for a in soup.find_all("a", href=True):
            href = a["href"]
            if href.lower().endswith(".pdf"):
                pdf_link = href
                break
        data["pdf_link"] = pdf_link

    except Exception as e:
        print("Error parsing HTML:", e)

    return data


def download_file_from_url(url: str, out_path: str):
    import requests
    try:
        resp = requests.get(url, stream=True, timeout=30)
        resp.raise_for_status()
        with open(out_path, "wb") as f:
            for chunk in resp.iter_content(1024 * 16):
                if chunk:
                    f.write(chunk)
        return True
    except Exception as e:
        print("Download failed:", e)
        return False


# -------------------- Cause list flow --------------------

def open_cause_list_page_and_get(driver, state: str = None, district: str = None, court_text: str = None, date_str: str = None):
    """
    Opens the cause list page and prompts the user to select the state/district/court and solve CAPTCHA.
    After user submits the form, the page source is returned.
    """
    url = "https://services.ecourts.gov.in/ecourtindia_v6/?p=cause_list%2Findex"
    driver.get(url)
    print("Opened Cause List page in browser. You will need to select State / District / Court and solve CAPTCHA in the page UI.")
    print("If you want, you can prefill fields in the browser manually after it opens.")
    input("After selecting the required items and submitting (and solving captcha if present), press Enter here to continue...")
    time.sleep(1)
    return driver.page_source


# -------------------- CLI & main --------------------

def build_argparser():
    p = argparse.ArgumentParser(description="eCourts Scraper — search CNR / case / download cause list")
    g = p.add_mutually_exclusive_group(required=True)
    g.add_argument("--cnr", help="Search by full CNR number (e.g. MHAU019999992015)")
    g.add_argument("--case", nargs=3, metavar=("TYPE","NUMBER","YEAR"), help="Search by Case Type, Number and Year. Example: CIVIL 123 2023")
    g.add_argument("--causelist", action="store_true", help="Open cause list UI to download cause list PDF")

    p.add_argument("--download-pdf", action="store_true", help="If case or cause list has a PDF available, download it")
    p.add_argument("--out", default="output", help="Output directory to save results")
    p.add_argument("--headless", action="store_true", help="Run browser headless (captcha still likely blocks flow)")
    return p


def main():
    args = build_argparser().parse_args()
    out_dir = ensure_output_dir(args.out)
    driver = make_driver(headless=args.headless)

    try:
        if args.cnr:
            cnr = args.cnr.strip()
            html = open_cnr_search_and_get(driver, cnr)
            parsed = parse_case_detail_html(html)
            parsed["queried_cnr"] = cnr
            # check whether next_hearing is today/tomorrow
            listed_flag = False
            when = None
            if parsed.get("next_hearing_raw"):
                is_listed, when = is_today_or_tomorrow(parsed["next_hearing_raw"])
                listed_flag = is_listed
            parsed["listed_today_or_tomorrow"] = listed_flag
            parsed["listed_when"] = when

            # Save JSON
            out_file = out_dir / f"case_{cnr}_{now_date_str()}.json"
            with open(out_file, "w", encoding="utf-8") as f:
                json.dump(parsed, f, indent=2, ensure_ascii=False)
            print(f"Saved parsed result to {out_file}")

            # optionally download pdf
            if args.download_pdf and parsed.get("pdf_link"):
                pdf_url = parsed["pdf_link"]
                if pdf_url.startswith("/"):
                    pdf_url = "https://services.ecourts.gov.in" + pdf_url
                fname = out_dir / Path(pdf_url).name
                ok = download_file_from_url(pdf_url, str(fname))
                if ok:
                    print(f"Downloaded PDF to {fname}")
                else:
                    print("Failed to download PDF. You may try copying the PDF URL into a browser.")

        elif args.case:
            ctype, cnum, cyear = args.case
            print("Searching by Case Type/Number/Year is supported via the eCourts UI. The script will open the search page; please fill any additional fields and solve CAPTCHA if required.")
            url = "https://services.ecourts.gov.in/ecourtindia_v6/?p=casestatus%2Findex"
            driver.get(url)
            # try to prefill fields if possible
            try:
                # attempt to find field names - these vary so best-effort
                try:
                    el = WebDriverWait(driver, 3).until(EC.presence_of_element_located((By.CSS_SELECTOR, "input[name='case_no']")))
                    el.clear(); el.send_keys(cnum)
                except Exception:
                    pass
                print("Please fill missing fields in the browser (if not auto-filled), solve CAPTCHA and submit the form. Then press Enter here.")
                input("Press Enter when done...")
                html = driver.page_source
                parsed = parse_case_detail_html(html)
                parsed["queried_case"] = {"type": ctype, "number": cnum, "year": cyear}
                # check hearings
                if parsed.get("next_hearing_raw"):
                    is_listed, when = is_today_or_tomorrow(parsed["next_hearing_raw"])
                    parsed["listed_today_or_tomorrow"] = is_listed
                    parsed["listed_when"] = when
                out_file = out_dir / f"case_{ctype}_{cnum}_{cyear}_{now_date_str()}.json"
                with open(out_file, "w", encoding="utf-8") as f:
                    json.dump(parsed, f, indent=2, ensure_ascii=False)
                print(f"Saved parsed result to {out_file}")
            except Exception as e:
                print("Error during case search flow:", e)

        elif args.causelist:
            print("Opening Cause List UI. Select State/District/Court, date and submit (solve CAPTCHA if shown).")
            html = open_cause_list_page_and_get(driver)
            # Attempt to find any PDF links (cause lists are frequently PDFs)
            soup = BeautifulSoup(html, "lxml")
            pdfs = []
            for a in soup.find_all("a", href=True):
                href = a["href"]
                if href.lower().endswith(".pdf"):
                    pdfs.append(href)
            if not pdfs:
                print("No PDF links automatically found on the resulting page. The cause list may be rendered as HTML or the PDF link is behind JS. If you see a link in your browser, copy its URL and paste it here to download it.")
                manual = input("Paste cause list PDF URL to download (or leave blank to skip): ").strip()
                if manual:
                    pdfs.append(manual)
            saved = []
            for pdf in pdfs:
                if pdf.startswith("/"):
                    pdf = "https://services.ecourts.gov.in" + pdf
                out_path = str(out_dir / Path(pdf).name)
                if download_file_from_url(pdf, out_path):
                    saved.append(out_path)
            result = {"found_pdfs": pdfs, "saved": saved}
            out_file = out_dir / f"causelist_{now_date_str()}.json"
            with open(out_file, "w", encoding="utf-8") as f:
                json.dump(result, f, indent=2, ensure_ascii=False)
            print(f"Saved cause list info to {out_file}")

    finally:
        print("Closing browser...")
        driver.quit()


if __name__ == "__main__":
    main()



