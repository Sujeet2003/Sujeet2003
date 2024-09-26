### Hi there 👋

<!--
**Sujeet2003/Sujeet2003** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->







import fitz  # PyMuPDF
import csv
import re
from pytesseract import image_to_string  # OCR for image-to-text extraction
from PIL import Image
import io

def extract_toc_from_text(pdf_path):
    """Extract Table of Contents from PDF text based on custom format."""
    toc = []
    text = extract_text_from_pdf(pdf_path)
    
    # Regex pattern to match TOC entries like "1. Hackathon Profile", "1.1 General Description", etc.
    toc_pattern = re.compile(r'(\d+(\.\d+)*\s+[^\n]+)')  # Matches "1. Title" or "1.1 Title"
    matches = toc_pattern.finditer(text)
    
    for match in matches:
        title = match.group(0).strip()
        page_start = text.find(title) + len(title)
        
        # Find the next title to determine the end of the current title's content
        next_match = next(matches, None)
        if next_match:
            page_end = text.find(next_match.group(0))
            page_number = text[page_start:page_end].count('\n') + 1  # Approximate page number
        else:
            page_number = text.count('\n')  # Last entry goes to last page

        # Append level and title to TOC
        level = title.count('.')  # Count the dots to determine the level
        toc.append((level + 1, title, page_number))

    return toc

def extract_text_from_pdf(pdf_path):
    """Extract text from a PDF file."""
    text = ""
    with fitz.open(pdf_path) as pdf:
        for page in pdf:
            text += page.get_text("text")
    return text

def get_content_by_page_range(pdf_path, start_page, end_page):
    """Extract content from a range of pages in the PDF, including text from images."""
    content = ""
    with fitz.open(pdf_path) as pdf:
        # Ensure valid page numbers
        if end_page is None or end_page > pdf.page_count:
            end_page = pdf.page_count

        if start_page < 0:
            start_page = 0

        for page_num in range(start_page, end_page):
            if page_num >= pdf.page_count:  # Check if the page number is valid
                break
            page = pdf[page_num]
            content += page.get_text("text")  # Extract the regular text

            # Extract and process images for OCR
            images = page.get_images(full=True)
            for img in images:
                xref = img[0]
                base_image = pdf.extract_image(xref)
                image_bytes = base_image["image"]
                image = Image.open(io.BytesIO(image_bytes))

                # Use OCR to extract text from the image
                ocr_text = image_to_string(image)

                if ocr_text.strip():  # Only add OCR text if something was recognized
                    content += f"\n[Image OCR Text]: {ocr_text}\n"

            # Extract tables from the page
            tables = extract_tables_from_page(page)
            content += "\n".join(tables)  # Append tables to content

    return content

def extract_tables_from_page(page):
    """Extract table data from a page."""
    tables = []
    # Assuming the tables are in a specific format, we can extract them as text.
    # This is a simplistic implementation; you may need to refine it based on your specific PDF structure.
    
    text = page.get_text("text")  # Get text from the page
    lines = text.split('\n')
    
    # Look for lines that contain tabular data. This is a simple heuristic.
    # Modify this logic based on your specific table structure.
    for line in lines:
        if re.search(r'\s+[-]+\s+', line):  # Check for lines with dashes (indicative of tables)
            tables.append(line.strip())

    return tables

def match_topic_with_content(toc, pdf_path):
    """Extract content for each topic and subtopic based on TOC and organize in a hierarchical structure."""
    topic_content = []
    parent_stack = []  # Stack to keep track of parent topics and sub-topics

    with fitz.open(pdf_path) as pdf:
        for i, (level, title, page) in enumerate(toc):
            # Update the stack based on the current level
            while len(parent_stack) >= level:  # Pop from stack if going to a higher level
                parent_stack.pop()
            parent_stack.append(title)  # Push current title onto the stack

            # Determine the end page (next topic's page minus 1) for the current topic
            if i + 1 < len(toc):
                next_page = toc[i + 1][2] - 1  # Next topic's starting page - 1
            else:
                next_page = pdf.page_count  # Last topic, extract till the end of the document

            # Ensure valid page numbers
            if page - 1 < 0 or page - 1 >= pdf.page_count or next_page >= pdf.page_count:
                print(f"Warning: Invalid page range for topic '{title}'. Skipping.")
                parent_stack.pop()  # Remove the last added title from stack
                continue

            # Extract content from the given page range, including OCR from images
            content = get_content_by_page_range(pdf_path, page - 1, next_page)  # Convert to 0-based index

            # Prepare an empty row with 11 elements (10 for sub-topic levels and 1 for content)
            row = [""] * 11

            for j in range(min(level, len(parent_stack))):  # Safely fill the row based on stack length
                row[j] = parent_stack[j]  # Fill the row with topic and sub-topic hierarchy

            row[10] = content.strip()  # Add content to the last column

            # Append row to topic_content
            topic_content.append(row)

    return topic_content

def store_in_csv(data, output_file):
    """Store extracted data in CSV format."""
    header = ['Topic', 'Sub-topic 1', 'Sub-topic 2', 'Sub-topic 3', 'Sub-topic 4', 'Sub-topic 5', 'Sub-topic 6', 'Sub-topic 7', 'Sub-topic 8', 'Sub-topic 9', 'Content']

    with open(output_file, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.writer(file)
        writer.writerow(header)
        for row in data:
            writer.writerow(row)

def main(pdf_path, output_file):
    # Extract the Table of Contents (TOC) using the custom method
    toc = extract_toc_from_text(pdf_path)

    if toc:
        print("Table of Contents found, extracting content based on TOC...")
        # Extract content based on TOC topics and subtopics
        topic_content = match_topic_with_content(toc, pdf_path)
    else:
        print("No Table of Contents found, extracting content using default method...")
        topic_content = []

    # Store the extracted content in a CSV format
    store_in_csv(topic_content, output_file)
    print(f"Data successfully stored in {output_file}.")

# Example usage
pdf_path = '/path/to/your/PDF.pdf'  # Replace with your PDF file path
output_file = 'output_topics_new.csv'
main(pdf_path, output_file)
