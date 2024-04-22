import pymqi
import cx_Oracle
import os
import logging
import glob
import uuid
from datetime import datetime
from lxml import etree

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Configuration for MQ
MQ_QUEUE_NAME = "YOUR_QUEUE_NAME"
MQ_CHANNEL = "YOUR_CHANNEL_NAME"
MQ_QUEUE_MANAGER = "YOUR_QUEUE_MANAGER"
MQ_HOST = "YOUR_MQ_HOST"
MQ_PORT = "YOUR_MQ_PORT"

# Configuration for Oracle DB
ORACLE_DSN = cx_Oracle.makedsn("YOUR_ORACLE_HOST", "YOUR_ORACLE_PORT", service_name="YOUR_SERVICE_NAME")
ORACLE_USER = "YOUR_ORACLE_USER"
ORACLE_PASSWORD = "YOUR_ORACLE_PASSWORD"

# Path to the folder with XML files
XML_FOLDER_PATH = "YOUR_XML_FOLDER_PATH"  # Update to your XML folder path

# Step 1: Modify XML files
def modify_xml_content(xml_content):
    # Parse the XML content
    root = etree.fromstring(xml_content)

    # Update the <UETR> tag with a unique UUID
    uetr_tag = root.find(".//UETR")
    if uetr_tag is not None:
        uetr_tag.text = str(uuid.uuid4())  # Unique UUID

    # Update the <PymtUid> tag with a unique 12-character string
    pymt_uid_tag = root.find(".//PymtUid")
    if pymt_uid_tag is not None:
        # Create a unique 12-character ID with 'M1' and timestamp (DDMMYYHHMM)
        timestamp_str = datetime.now().strftime("%d%m%y%H%M")
        unique_pymt_uid = f"M1{timestamp_str}"
        pymt_uid_tag.text = unique_pymt_uid

    # Return the modified XML as a string
    return etree.tostring(root)

# Step 2: Connect to MQ and inject modified messages
try:
    qmgr = pymqi.connect(MQ_QUEUE_MANAGER, MQ_CHANNEL, MQ_HOST + f"({MQ_PORT})")
except Exception as e:
    logging.error("Error connecting to MQ: %s", str(e))
    raise

try:
    xml_files = glob.glob(os.path.join(XML_FOLDER_PATH, "*.xml"))

    if not xml_files:
        raise FileNotFoundError(f"No XML files found in folder: {XML_FOLDER_PATH}")

    queue = pymqi.Queue(qmgr, MQ_QUEUE_NAME)

    for xml_file in xml_files:
        with open(xml_file, 'rb') as file:
            original_content = file.read()  # Read original XML content
            modified_content = modify_xml_content(original_content)  # Modify XML content
            queue.put(modified_content)  # Inject modified content into MQ
            logging.info("Injected modified message from %s into MQ", xml_file)

    queue.close()
    qmgr.disconnect()

except Exception as e:
    logging.error("Error injecting messages into MQ: %s", str(e))
    raise

# Step 3: Connect to Oracle DB to verify modified messages
try:
    with cx_Oracle.connect(ORACLE_USER, ORACLE_PASSWORD, ORACLE_DSN) as connection:
        with connection.cursor() as cursor:
            # Customize this query to check if the messages were processed
            # Modify 'YOUR_TABLE' and 'YOUR_COLUMN' with your table and validation criteria
            query = "SELECT * FROM YOUR_TABLE WHERE YOUR_COLUMN = :value"

            for xml_file in xml_files:
                # Read the original file again and apply the same modifications
                with open(xml_file, 'r') as file:
                    original_content = file.read()
                    modified_content = modify_xml_content(original_content).decode("utf-8")

                cursor.execute(query, {"value": modified_content.strip()})
                result = cursor.fetchone()

                if result:
                    logging.info("Verified modified message from %s in Oracle DB: %s", xml_file, result)
                else:
                    logging.warning("Modified message from %s not found in Oracle DB", xml_file)

except cx_Oracle.Error as e:
    logging.error("Error connecting to Oracle DB or executing query: %s", str(e))
    raise
