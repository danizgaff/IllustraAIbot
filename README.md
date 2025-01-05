import logging
from telegram import Update, InputFile
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
import requests
from io import BytesIO

# Configure logging
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)  # Use the `__name__` variable for the logger

# API keys
TELEGRAM_BOT_TOKEN = "7813381356:AAGwlqKvDlcg8GIkaqLX-P8wlt0ursvl0CI"  # Your Telegram Bot Token
HF_API_KEY = "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"  # Your Hugging Face API key

# Hugging Face API URL
HF_API_URL = "https://api-inference.huggingface.co/models/stabilityai/stable-diffusion-2"


# Function to generate an image using Hugging Face API
def generate_image_from_huggingface(prompt: str) -> BytesIO:
    """Generate an image using Hugging Face's API."""
    headers = {"Authorization": f"Bearer {HF_API_KEY}"}
    payload = {"inputs": prompt}

    # Send the request to the Hugging Face API
    response = requests.post(HF_API_URL, headers=headers, json=payload)

    if response.status_code == 200:
        # Return the image as a BytesIO object
        return BytesIO(response.content)
    else:
        logger.error(f"Hugging Face API error: {response.status_code}, {response.text}")
        raise Exception("Failed to generate image. Check your prompt or API key.")


# Command: Start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Send a welcome message when the /start command is issued."""
    await update.message.reply_text(
        "Hello! I am an AI-powered image generator bot. Use /genim to generate an image based on your description.\n"
        "\nCommands:\n"
        "/start - Start the bot\n"
        "/genim - Generate an AI image"
    )


# Command: Generate Image
async def generate_image(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Prompt the user to enter a description for the AI image."""
    await update.message.reply_text("Please send me a description for the image you'd like to generate.")
    return 1  # Wait for the user's message


# Generate Image Based on User Prompt
async def generation_with_prompt(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Generate an AI image based on the user's prompt."""
    user_prompt = update.message.text

    # Inform the user that generation is in progress
    await update.message.reply_text("Creating your image... Please wait.")

    try:
        # Generate the image using Hugging Face API
        image_data = generate_image_from_huggingface(user_prompt)

        # Send the image back to the user
        await update.message.reply_photo(
            photo=InputFile(image_data, filename="ai_image.png"),
            caption="Here's your AI-generated image!"
        )
    except Exception as e:
        logger.error(f"Error during image generation: {e}")
        await update.message.reply_text("Sorry, I couldn't generate the image. Please try again later.")


# Error Handler
async def error(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Log errors caused by updates."""
    logger.warning(f"Update {update} caused error {context.error}")


# Main Function to Run the Bot
def main():
    """Run the bot."""
    application = Application.builder().token(TELEGRAM_BOT_TOKEN).build()

    # Add command handlers
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("genim", generate_image))

    # Add message handler for the next step
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, generation_with_prompt))

    # Add error handler
    application.add_error_handler(error)

    # Start the bot
    application.run_polling()


# Run the bot if the script is executed directly
if __name__ == "__main__":
    print("Starting the bot...")
    main()

