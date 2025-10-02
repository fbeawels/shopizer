# Shopizer Content Management Interface

## Summary
This image depicts an administrative interface of the Shopizer e-commerce platform, specifically focusing on the content management section. The interface is designed to manage various aspects of content pages within the system.

## Detailed Description

### Overall Layout and Structure
The interface follows a typical admin panel layout with a sidebar navigation on the left and a main content area on the right. The design is clean and functional, prioritizing usability over aesthetics.

### Sidebar Navigation
- **Position**: Left side of the screen
- **Width**: Approximately 20% of the screen width
- **Components**:
  - **Home**: Topmost option with a house icon
  - **User Management**: Second option with a user icon
  - **Store Management**: Third option with a store icon
  - **Inventory Management**: Fourth option with an inventory icon
  - **Content Management**: Fifth option, currently expanded, with a content icon
    - **Content Pages**: Currently selected sub-option
    - **Content Boxes**
    - **Images**
  - **Shipping Management**: Sixth option with a shipping icon
  - **Payment**: Seventh option with a payment icon
  - **Tax Management**: Eighth option with a tax icon
  - **Customer Management**: Ninth option with a customer icon
  - **Order Management**: Tenth option with an order icon

### Main Content Area
- **Position**: Right side of the screen
- **Width**: Approximately 80% of the screen width
- **Components**:
  - **Header**: Contains the title "COMPONENTS.CONTENT_PAGES"
  - **Store Selection Dropdown**: Labeled "STORE.MERCHANT_STORE"
  - **Create Page Button**: Labeled "CONTENT.CREATE_PAGE" with a plus icon
  - **Content Table**:
    - **Columns**:
      - **COMMON.ID**
      - **CONTENT.CODE**
      - **CONTENT.NAME**
      - **CONTENT.URL**
      - **ORDER.ACTIONS**
    - **Status**: Currently empty with the message "No data found"

### Visual Elements
- **Colors**:
  - Primary background: Light gray
  - Sidebar background: Darker gray
  - Text: Black
  - Icons: Blue
- **Styles**:
  - Clean, minimalistic design
  - Consistent use of icons for visual cues
  - Clear separation between different sections

### Text Content and Formatting
- **Text**: Primarily in English
- **Formatting**:
  - Headers are bold and capitalized
  - Table headers are bold
  - Consistent use of uppercase for labels and buttons

### Distinctive Features
- **Language Selection**: Located at the top right corner, labeled "Languages - (English)"
- **User Role**: Indicated as "Administrator User" at the top right corner
- **Footer**: Contains copyright information "© Shopizer 2010-2025" and a URL "localhost:8082/#/pages/content/pages/list"

### Notable Observations
- The interface is designed for ease of navigation with a clear hierarchy of options.
- The use of icons alongside text labels enhances visual recognition and usability.
- The empty state of the content table suggests that no content pages have been created yet, or the filter applied does not match any existing content pages.