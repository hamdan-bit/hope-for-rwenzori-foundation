# Use official Node.js image
FROM node:18

# Set working directory inside container
WORKDIR /app

# Copy package.json and package-lock.json first
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of your source code
COPY . .

# Expose the port your Express app runs on
EXPOSE 3000

# Start the app
CMD ["npm", "start"]

# Copy .env file
COPY .env .env
