# Base image
FROM node:18

# Set working directory
WORKDIR /app

# Set environment variables
ENV PORT=3000
ENV MONGO_URI=mongodb://root:example@mongo:27017/tododb?authSource=admin

# Copy files
COPY package*.json ./
RUN npm install

COPY . .

# Expose port
EXPOSE 3000

# Start the app
CMD ["npm", "start"]

