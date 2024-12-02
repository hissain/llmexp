```
import requests
import json
import os
import re
from fpdf import FPDF
from markdown_pdf import MarkdownPdf

# Ollama API configuration
OLLAMA_URL = "http://107.109.10.95:11434/api/generate"
MODEL_NAME = "llama3.2:latest"

# Directory to save generated content
OUTPUT_DIR = "book_content_large"
os.makedirs(OUTPUT_DIR, exist_ok=True)

def generate_toc(subject):
    """Generate a Table of Contents for the given subject."""
    prompt = f"""
    You are an expert assistant skilled at creating detailed book outlines.
    Given a subject, your task is to create a comprehensive Table of Contents for a book on the given subject.
    
    Each chapter should have sections, and when relevant, sections should have subsubsections. 
    Structure the Table of Contents hierarchically to ensure maximum detail and granularity.
    
    Now generate a detailed Table of Contents for the book with Subject: {subject}.
    
    Structure the response as a valid JSON object:
    {{
        "Table_of_Contents": [
            {{
                "Title": "Chapter 1: Title of Chapter",
                "Sections": [
                    {{
                        "Title": "Section 1.1: Title of Section",
                        "Subsections": [
                            "Subsubsection 1.1.1: Title of Subsubsection",
                            "Subsubsection 1.1.2: Title of Subsubsection"
                        ]
                    }},
                    {{
                        "Title": "Section 1.2: Title of Section",
                        "Subsections": []
                    }}
                ]
            }},
            {{
                "Title": "Chapter 2: Title of Chapter",
                "Sections": []
            }}
        ]
    }}
    """
    payload = {"model": MODEL_NAME, "prompt": prompt, "stream": False}
    response = requests.post(OLLAMA_URL, json=payload)
    
    if response.status_code != 200:
        raise Exception(f"Error from Ollama: {response.text}")
    
    content = response.json().get("response", "")
    # Extract JSON using regex
    match = re.search(r"\{.*\}", content, re.DOTALL)
    if match:
        json_content = match.group(0)
        try:
            return json.loads(json_content)
        except json.JSONDecodeError as e:
            raise Exception(f"Failed to parse JSON: {e}")
    else:
        raise Exception(f"No JSON object found in response: {content}")

def generate_section(title):
    """Generate content for a given chapter, section, or subsubsection title."""
    prompt = f"""
    You are a knowledgeable assistant tasked with writing detailed content for a book.
    Please write content for the following title:
    Title: '{title}'
    
    The content should be comprehensive, well-structured, and professional.
    Include examples, subtopics, and technical insights where applicable. 
    Expand on subtopics wherever possible to provide a deep understanding of the subject.
    """
    payload = {"model": MODEL_NAME, "prompt": prompt, "stream": False}
    response = requests.post(OLLAMA_URL, json=payload)
    
    if response.status_code != 200:
        raise Exception(f"Error from Ollama: {response.text}")
    
    return response.json().get("response", "")

def save_content_to_file(filename, content):
    """Save generated content to a file."""
    with open(os.path.join(OUTPUT_DIR, filename), "w") as f:
        f.write(content)

import re

def parse_markdown_to_pdf(pdf, content):
    """Parse and apply Markdown-like formatting to content for PDF."""
    content = content.encode('utf-8', errors='ignore').decode('latin-1')
    lines = content.split('\n')
    for line in lines:
        pdf.set_font("Arial", size=10)
        if line.startswith('### '):  # Heading level 3
            pdf.set_font("Arial", size=12, style="B")
            pdf.cell(0, 5, line[4:])
        elif line.startswith('## '):  # Heading level 2
            pdf.set_font("Arial", size=14, style="B")
            pdf.cell(0, 5, line[3:])
        elif line.startswith('# '):  # Heading level 1
            pdf.set_font("Arial", size=16, style="B")
            pdf.cell(0, 5, line[2:])
        elif "**" in line:
            line = re.sub(r'^\d+\.\s(.*)$', r'\1', line, flags=re.MULTILINE)
            bold_parts = re.split(r"(\*\*.*?\*\*)", line)
            for part in bold_parts:
                if len(part.strip()) != 0:
                    if part.startswith("**") and part.endswith("**"):
                        pdf.set_font("Arial", style="B")
                        pdf.multi_cell(0, 5, part[2:-2])
                    elif part.startswith(': '):
                        pdf.set_font("Arial", size=10)
                        pdf.multi_cell(0, 5, part[2:])
                    elif part.strip().startswith('*'):
                        pdf.cell(10, 5, "", ln=False)
                    else:
                        pdf.set_font("Arial", size=10)
                        pdf.multi_cell(0, 5, part)
        else:
            pdf.set_font("Arial", size=10)
            pdf.multi_cell(0, 5, line, border=0)


def create_pdf_from_content(toc, output_file=f"{OUTPUT_DIR}/Generated_Book.pdf"):
    """Generate a PDF book from the content."""
    pdf = FPDF()
    pdf.set_auto_page_break(auto=True, margin=15)
    pdf.set_left_margin(15)
    pdf.set_right_margin(15)
    
    # Title Page
    pdf.add_page()
    pdf.set_font("Arial", size=22, style="B")
    pdf.cell(0, 10, subject, ln=True, align="C")
    pdf.ln(20)
    
    # Table of Contents
    pdf.set_font("Arial", size=14, style="B")
    pdf.cell(0, 10, "Table of Contents", ln=True)
    pdf.ln(10)
    pdf.set_font("Arial", size=10)
    for chapter in toc["Table_of_Contents"]:
        pdf.cell(0, 5, chapter["Title"], ln=True)
        for section in chapter["Sections"]:
            pdf.cell(20, 5, f"  {section['Title']}", ln=True)
            for subsubsection in section["Subsections"]:
                pdf.cell(30, 5, f"    {subsubsection}", ln=True)
        pdf.ln(5)
    
    # Chapters
    for chapter in toc["Table_of_Contents"]:
        pdf.add_page()
        pdf.set_font("Arial", size=14, style="B")
        pdf.cell(0, 5, chapter["Title"], ln=True)
        pdf.ln(10)
        
        chapter_file = f"{chapter['Title'].replace(' ', '_')}.txt"
        chapter_path = os.path.join(OUTPUT_DIR, chapter_file)
        if os.path.exists(chapter_path):
            with open(chapter_path, "r") as f:
                content = f.read()
            parse_markdown_to_pdf(pdf, content)
        
        for section in chapter["Sections"]:
            pdf.add_page()
            pdf.set_font("Arial", size=14, style="B")
            pdf.cell(0, 5, section["Title"], ln=True)
            pdf.ln(5)
            
            section_file = f"{section['Title'].replace(' ', '_')}.txt"
            section_path = os.path.join(OUTPUT_DIR, section_file)
            if os.path.exists(section_path):
                with open(section_path, "r") as f:
                    content = f.read()
                parse_markdown_to_pdf(pdf, content)
            
            for subsubsection in section["Subsections"]:
                pdf.add_page()
                pdf.set_font("Arial", size=12, style="B")
                pdf.cell(0, 5, subsubsection, ln=True)
                pdf.ln(5)
                
                subsubsection_file = f"{subsubsection.replace(' ', '_')}.txt"
                subsubsection_path = os.path.join(OUTPUT_DIR, subsubsection_file)
                if os.path.exists(subsubsection_path):
                    with open(subsubsection_path, "r") as f:
                        content = f.read()
                    parse_markdown_to_pdf(pdf, content)
    
    pdf.output(output_file)

# Main Execution
if __name__ == "__main__":
    subject = "Large Language Models"
    print(f"Generating Table of Contents for subject: {subject}")
    
    # Step 1: Generate ToC
    #toc = generate_toc(subject)
    print("Table of Contents generated successfully.")
    save_content_to_file(f"toc.txt", json.dumps(toc, indent=4))
    
    # Step 2: Generate content for each chapter/section/subsubsection
    for chapter in toc["Table_of_Contents"]:
        chapter_title = chapter["Title"]
        print(f"Generating content for {chapter_title}...")
        #chapter_content = generate_section(chapter_title)
        #save_content_to_file(f"{chapter_title.replace(' ', '_')}.txt", chapter_content)
        
        for section in chapter["Sections"]:
            section_title = section["Title"]
            print(f"Generating content for {section_title}...")
            #section_content = generate_section(section_title)
            #save_content_to_file(f"{section_title.replace(' ', '_')}.txt", section_content)
            
            for subsubsection_title in section["Subsections"]:
                print(f"Generating content for {subsubsection_title}...")
                #subsubsection_content = generate_section(subsubsection_title)
                #save_content_to_file(f"{subsubsection_title.replace(' ', '_')}.txt", subsubsection_content)
    
    # Step 3: Create PDF
    print("Merging content into a PDF book...")
    #create_pdf_from_content(toc)
    create_pdf_from_content(toc)
    print(f"PDF book generated successfully at location: {OUTPUT_DIR}/Generated_Book_Detail.pdf")
```
